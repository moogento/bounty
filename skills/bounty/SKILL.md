---
name: bounty
description: This skill should be used when the user asks to "run a bounty", "bug bounty", "start a bounty hunt", "hunt bugs", "find bugs", "scan for bugs", or wants to discover unknown bugs across a codebase using multiple competing agents. Use when the goal is discovery (unknown bugs) rather than fixing a specific known bug (for that, use zap or multiagent-fix).
argument-hint: "[--scope <path>] [--agents N] [--severity low|medium|high|critical] [--max-claims N] [--no-fix]"
---

# Bounty

Discover bugs across a codebase by running competing specialist agents in parallel. Agents hunt, vote on each other's claims, fix confirmed bugs, and earn points. False positives are penalized so shallow claims lose the round.

Use this for **discovery** of unknown bugs. For fixing a *known* bug, use `zap` or `multiagent-fix` instead.

## Arguments

Parse `$ARGUMENTS` for these flags (all optional):

| Flag | Default | Meaning |
|------|---------|---------|
| `--scope <path>` | repo root | Restrict hunt to a directory or glob |
| `--agents N` | 8 | Number of specialist lenses in play (min 3, max 12) |
| `--hunters-per-lens K` | 1 | Hunters dispatched per lens — set ≥2 to treat independent collisions as validation |
| `--severity <floor>` | `medium` | Discard claims below this severity |
| `--max-claims N` | 40 | Cap total claims to keep cost bounded |
| `--no-fix` | false | Skip Phase 4–6, just report findings |
| `--voting-mode <mode>` | `auto` | `full` = N−1 voters per claim (small runs). `panel` = 1 voter per specialty reviews every non-own claim (8 agents total, independent of claim count). `auto` = `full` when claims ≤ 20, `panel` otherwise |
| `--max-fixes N` | 10 | If confirmed count exceeds this, Phase 4 pauses and asks the user to confirm/curate/reject the bundle plan before dispatching planners |
| `--model-hunt <tier>` | `sonnet` | Model tier for hunters |
| `--model-vote <tier>` | `haiku` | Model tier for voters; orchestrator auto-falls back to `sonnet` on haiku rate-limit errors (announced) |
| `--model-plan <tier>` | `opus` | Model tier for bundle planners |
| `--model-fix <tier>` | `sonnet` | Model tier for bundle implementers |
| `--max-bundles N` | 6 | Cap on Phase-4 bundles. Smallest lenses merge into a `misc` bundle when exceeded |

If no scope is given, ask the user once whether to scope the hunt.

## Phase 0: Initialize state

Create the state directory and seed files:

```bash
mkdir -p .temp/bounty/{claims,votes,fixes,plans,reviews}
```

**Resolve and record the absolute state directory path.** Agents dispatched in isolated worktrees have their own `.temp/bounty/` — a relative path will silently write into the worktree instead of the main repo's state dir. Compute once at Phase 0 and inject into every agent prompt as `STATE_DIR`:

```bash
STATE_DIR="$(cd .temp/bounty && pwd)"
```

Write `.temp/bounty/config.json` with the resolved arguments *and* `state_dir: "<absolute path>"`. Every downstream agent prompt must include a `STATE_DIR=<absolute>` line and instructions to write every output file under `$STATE_DIR/...` — never a relative `.temp/bounty/...` path.

Write an empty `.temp/bounty/leaderboard.json` with zeroed scores for the chosen specialists. See `references/state-schema.md` for file schemas.

**Probe the voter-tier quota.** Before dispatching hunters, fire one throwaway haiku call (or whatever `$MODEL_VOTE` resolves to) and catch a `rate-limit` / `usage limit` error. If it fires, warn the user at Phase 0:

```
⚠ voter-tier model (haiku) returned a rate-limit on probe; Phase 3 will fall back to sonnet per claim. Expect higher voter cost.
```

This prevents the Phase 3 surprise where 8 haiku voters all fail on their first dispatch.

## Live progress output

The orchestrator must surface progress **as events happen**, not just at phase boundaries. The user is watching — silent phases feel broken.

Rules that apply to every phase below:

1. **Tail state directories while agents run.** Hunters, voters, and fixers all write files as their source of truth. Between dispatch and final `TaskList` wait, run short polling Bash calls (≈20–30s apart, max ~20 iterations per phase) that list new files and print a one-liner per file.
2. **Print a one-liner per event as it lands.** Use compact, scannable formats:
   - New claim: `🛡️ Security/1 → BUG-003  high  sql-injection  app/Http/Orders.php:142`
   - Collision: `🛡️ Security/2 collided with 🛡️ Security/1 on BUG-003  (collision_count = 2)`
   - New vote: `BUG-003  ✓ VALID  from 🧵 Concurrency  (sanitizer bypassed upstream)`
   - Claim resolved: `BUG-003 CONFIRMED (4 VALID / 1 FP / 2 abstain)  finder +1 → 🛡️ Security`
   - Bundle planned: `📦 fix-acl  3 bugs  test_strategy=tdd  reviewer=haiku`
   - Plan written: `📋 plans/fix-acl.md written  3 bugs  commit_order=[…]`
   - Fix committed: `BUG-003 (fix-acl) committed by ⚙ implementer-fix-acl  (2 files, test added, phpunit ✓)`
   - Bundle reviewed: `fix-acl  3/3 APPROVE from 🧵  → kept, implementer +3`
3. **Re-render the running leaderboard on every state change** (new confirmation, new FP, new fix-approval). Print the same table format as `references/scoring-rules.md`, prefixed with `— leaderboard @ <phase> —`.
4. **Never batch.** If three claims land during one poll tick, print three lines — do not summarize as "3 new claims."
5. Keep the poll loop bounded: exit as soon as `TaskList` shows all dispatched agents in that phase complete, or after the iteration cap (then fall through to the phase's final wait).

## Phase 1: Shared recon

Spawn **1 recon agent** in an isolated worktree (`isolation: "worktree"`, `model: "sonnet"`, `run_in_background: false`). Its job: produce a shared briefing that every hunter receives as cached context.

**Scale the time budget by scope size**:

```
files_in_scope = count of non-vendor files under --scope
budget_minutes = max(5, ceil(files_in_scope / 100))
```

So a 120-file module gets 5 minutes, a 500-file module gets 5 minutes, a 1200-file repo gets 12, a 3000-file monorepo gets 30. Record the chosen budget in `.temp/bounty/config.json` as `recon_budget_minutes`.

Prompt must include:
- The scope (or "repo root")
- `STATE_DIR=<absolute path>` (see Phase 0)
- Instructions to:
  - Map the major modules and their responsibilities
  - Identify external boundaries (HTTP, DB, filesystem, third-party APIs, user input)
  - Run `git log --since="90 days ago" --stat` on the scope and note high-churn files
  - List test coverage gaps (files with no tests in the scope)
  - Return a **markdown report** (≤2k words) with sections: Modules, Boundaries, Recent churn, Coverage gaps, Suspicion heatmap
- The computed time budget (pass the integer minutes, do not hardcode 5)

Save the report to `$STATE_DIR/recon.md`. This is the shared briefing — pass it verbatim to every downstream agent so the prompt cache reuses it.

## Phase 2: The hunt (parallel specialists)

Assign each of N agents one **specialist lens** from `references/agent-specialties.md`. Any agent can still claim any bug (that is the rule), but the lens is their starting point to reduce pure overlap.

When `--hunters-per-lens K > 1`, dispatch **K hunters per lens** as a single flat parallel wave (total = N×K agents). Give each hunter within a lens a distinct **prompt seed** so they do not collapse into the same search path:
- seed 1: "start at the top of the module tree; breadth first"
- seed 2: "start at recent churn from the recon report; follow change-prone files"
- seed 3: "start at test-coverage gaps from the recon report; hunt where tests don't look"

Collisions between hunters on the same lens are the *point* — two independent hunters reaching the same bug is the pre-vote validation signal.

**Dispatch all N×K hunters in a single message**, each with:
- `isolation: "worktree"`
- `name: "hunter-<lens>-<seed>"` (so they are addressable later; seed defaults to `1` when K=1)
- `model: "sonnet"` (or `$MODEL_HUNT`) — **all** hunters run at the hunt tier regardless of seed index, since the second and third hunters exist to cheaply cross-check, not to duplicate at a higher tier
- `run_in_background: true`
- `subagent_type: "general-purpose"` (unless a specialist subagent fits — e.g. `moo-security-reviewer` for the security lens)

Each prompt must include:
- Their specialist lens with 3–5 concrete patterns to watch for (pulled from `references/agent-specialties.md`)
- Their prompt seed (if K>1) — phrased as the starting point, not a box they cannot leave
- The scope
- `STATE_DIR=<absolute path>` (see Phase 0). Every claim file must be written as `$STATE_DIR/claims/BUG-<lens>-<seed>-<seq>.json` with the absolute path — never a relative `.temp/bounty/...` which silently lands in the worktree
- The recon report, inlined verbatim (not referenced by path — worktree agents can't read the main repo's path)
- The severity floor
- A **tool-call budget**: cap `Read` calls at 20 and `Grep`/`Glob` calls at 15. If they need to read more, they've scoped too widely and should stop and submit what they have. This prevents the "Prompt is too long" blowout that kills hunters at 100+ tool calls.
- Instructions to:
  - Scan their lens from the prompt-seed entry point first, then broaden
  - For each claim, verify the evidence by reading the code — do not guess
  - Discard claims below the severity floor
  - **Write each claim to its JSON file the moment it is verified**, not in a batch at the end — the orchestrator tails this directory for live output. Path: `$STATE_DIR/claims/BUG-<lens>-<seed>-<seq>.json` (see `references/state-schema.md`). Never emit claims only to stdout — the file is the source of truth.
  - **Hard stop** the moment 5 claims are filed — do not keep exploring "just in case". Further reading after claim 5 is wasted budget and risks prompt-length death before the summary can be returned.
  - Or stop at 10 minutes elapsed, whichever comes first
  - Return a final one-line summary: `<lens>/<seed>: <n> claims submitted`

### Live tailing while hunters run

Do not simply block on `TaskList`. Instead, alternate short poll cycles until all hunters complete:

1. Track a set of claim filenames already printed.
2. Every ~25s run a Bash call like `ls -t .temp/bounty/claims/*.json 2>/dev/null` and diff against the printed set.
3. For each new file, read it and print the **new-claim one-liner** from the Live progress rules. Also print a compact running count: `claims so far: 🛡️ 3 · 🧵 2 · ⚡ 1 · … (total 6)`.
4. Call `TaskList` at the end of each cycle; when every hunter is `completed`, break.

Then read the claims directory in full and proceed.

### Deduplicate and count collisions

Compute a fingerprint for each claim: `sha256(normalized_file + ":" + category + ":" + line_bucket)` where `line_bucket = line // 10`. Claims with the same fingerprint are duplicates — keep the first-submitted, attribute the later ones as "co-discovered" (no bonus, no penalty).

On every merge, increment `collision_count` on the surviving claim and append the secondary finder (`<lens>/<seed>`) to `co_discovered_by`. Print a collision one-liner the moment it happens:

```
🛡️ Security/2 collided with 🛡️ Security/1 on BUG-security-1-003  (collision_count = 2)
```

Surface `collision_count` in the live leaderboard as a confidence column; pre-validated claims (`collision_count ≥ 2`) still go through Phase 3 voting unchanged, but reviewers and planners downstream can weight them.

### Secondary semantic-dedup pass

The fingerprint above misses cross-specialty semantic duplicates — two specialists can file the same underlying bug under different category labels (e.g. security/"Broken Access Control" vs authz/"ACL resource mismatch" on the same controller line). After the primary pass, run a **secondary dedup** over surviving claims:

- Bucket claims by `(normalize(file), line // 10)` ignoring category
- For any bucket with ≥2 claims from different specialties, flag the group as a semantic-duplicate candidate
- The orchestrator merges: keep the highest-severity claim as survivor, append the others' finders to `co_discovered_by`, increment `collision_count`, and record the merged categories in `merged_categories`
- Print a semantic-merge one-liner:

```
🔗 semantic merge: BUG-authz-1-001 + BUG-security-1-004 → surviving BUG-security-1-004  (ACL on same controller line)
```

Unlike exact-fingerprint collisions, semantic merges are *suggestive* — the orchestrator should log them and proceed. If a voter later disputes the merge (VALID but "different bug"), the orchestrator can split them back apart, but by default merged wins.

### Enforce `--max-claims`

If total claims exceed the cap, keep the highest-severity claims and drop the rest. Log the drop count.

## Phase 3: Parallel voting

### Voting modes

| Mode | Dispatch pattern | Agent runs | When to use |
|------|------------------|-----------|-------------|
| `full` | For each claim, dispatch N−1 voters in parallel | `claims × (N−1)` | Small runs (≤20 claims) — independent verdicts per claim, no cross-contamination |
| `panel` | Dispatch **1 voter per specialty**; each reviews every claim not authored by their own lens and appends all verdicts to a single `$STATE_DIR/votes/VOTER-<lens>.jsonl` | N (e.g. 8) | Large runs (>20 claims) — at 33 claims `full` would mean 231 agent runs; `panel` is 8. Observed: no loss of quality on the first production run |
| `auto` | Resolve to `full` if `surviving_claims ≤ 20`, else `panel` | varies | Default. Announce the resolved mode at Phase 3 start |

Resolve the mode from `--voting-mode` and print:

```
Voting mode: panel (33 claims > 20 threshold — dispatching 8 voters)
```

### Panel-mode dispatch

Dispatch all N panel voters in a single message. Each voter's prompt gets the full list of claims (JSON objects, cited excerpts inline) *except those filed by the voter's own lens* and writes verdicts to one JSONL file:

```
$STATE_DIR/votes/VOTER-<lens>.jsonl
```

Each line: `{"bug_id": "<id>", "verdict": "VALID|FALSE_POSITIVE|ABSTAIN", "reason": "<one sentence>", "voter": "<lens>", "at": "<iso>"}`.

Tally by reading every `VOTER-*.jsonl` file and grouping lines by `bug_id`. Write the aggregate to `$STATE_DIR/tally.json`.

### Full-mode dispatch

For each surviving claim, the **other N−1 agents** vote. Dispatch all voters for a given claim **in a single message** using `model: $MODEL_VOTE` (default `haiku`).

Each voter prompt includes:
- The claim JSON
- The file excerpt cited as evidence (read it, include ±10 lines)
- `STATE_DIR=<absolute>`
- Instructions: **append the verdict line to `$STATE_DIR/votes/<BUG-ID>.jsonl` immediately upon deciding** — do not hold it for a batch return. The orchestrator tails this file for live output. Format: `{"verdict": "VALID"|"FALSE_POSITIVE"|"ABSTAIN", "reason": "<one sentence>", "voter": "<lens>"}`

Process claims in batches of 5 to keep parallelism bounded.

### Voter model fallback

If any voter call returns a rate-limit / usage-cap error (common with haiku at scale — observed: 8 concurrent haiku voters all rejected on first dispatch):

1. Announce once: `⚠ haiku rate-limited; falling back to sonnet for remaining voters this phase`
2. Persist the fallback in `.temp/bounty/config.json` as `model_vote_effective: "sonnet"` so resumed runs don't re-probe
3. Retry the failed voter with sonnet
4. Keep going — do not hard-fail the phase on a voter-tier rate limit

### Live tailing during voting

Between dispatching a batch and it finishing, alternate ~20s poll cycles:

1. For each `<BUG-ID>.jsonl` in `.temp/bounty/votes/`, compare its current line count to the count printed last cycle. Print the **new-vote one-liner** for each newly appended line.
2. When a claim's vote count reaches N−1 (all voters in), apply the resolution rules immediately and print the **claim-resolved one-liner** plus an updated leaderboard table (see Live progress output rules).
3. Break when every claim in the batch is resolved, then proceed to the next batch.

### Resolution rules

Read the tallied votes and apply:

| Outcome | Condition | Points |
|---------|-----------|--------|
| CONFIRMED | majority VALID among non-abstaining voters | finder **+1** |
| FALSE_POSITIVE | ≥⌈(N−1)/3⌉ FALSE_POSITIVE votes **and** majority is FALSE_POSITIVE | finder **−3** |
| INCONCLUSIVE | neither | 0 |

Update `.temp/bounty/leaderboard.json` atomically. Update `.temp/bounty/README.md` with the current leaderboard (see `references/scoring-rules.md` for format).

## Phase 4: Bundled fixes (skip if `--no-fix`)

Confirmed bugs enter `.temp/bounty/queue.json` with their severity. Instead of dispatching one fixer per bug (which scales poorly when 30+ bugs land), Phase 4 runs a **plan → implement → review** pipeline over **bundles** of related bugs.

### Volume gate

Before dispatching any planners, check `confirmed_count` against `--max-fixes` (default 10). If `confirmed_count > max_fixes`, **ping the user with the bundle plan and wait up to 3 minutes** — do not block indefinitely. Auto-proceed with all bundles on timeout so the run can finish unattended.

```
🔔 33 confirmed bugs > --max-fixes=10. Auto-proceeding with all bundles in 3 min unless you reply.

Bundle plan:
  📦 fix-acl              3 bugs   [authz]         tdd?
  📦 fix-fail-open        5 bugs   [errorhandling] tdd?
  📦 fix-nullsafety-misc  7 bugs   [nullsafety + security]  tdd?
  📦 fix-concurrency      5 bugs   [concurrency]   architectural?
  📦 fix-data-integrity   5 bugs   [dataintegrity] architectural?
  📦 fix-resources-perf   8 bugs   [performance + resources] architectural?

Reply with:
  (a) all — proceed with every bundle (default after 3 min)
  (b) high-only — drop medium/low bundles
  (c) pick — list bundle slugs to keep, comma-separated
  (d) abort — stop after findings, no fixes
```

Send a `PushNotification` alongside the message so the user sees the ping even if the terminal isn't focused. Then wait up to 180 seconds for a reply. On reply: honor the choice and record `user_fix_scope` in `config.json`. On timeout: record `user_fix_scope: "all"` with `gate_timeout: true`, announce `"no reply in 3 min — proceeding with all bundles"`, and continue. Skip the gate entirely when `confirmed_count ≤ max_fixes`.

### Severity policy

| Severity | Action |
|----------|--------|
| low | Fix + commit inside a bundle |
| medium | Fix + commit inside a bundle |
| high | Fix + commit inside a bundle + flag for human review in PR body |
| critical | Do **not** auto-fix. Open a GitHub issue with full evidence via `gh issue create` |

### Step 4a: Bundle confirmed bugs

Cluster fixable confirmed bugs into **bundles** and write `.temp/bounty/bundles.json`. Default heuristic:

1. Start with one bundle per specialist lens that has ≥2 confirmed bugs.
2. Merge lenses with a single confirmed bug into a `misc` bundle for adjacent lenses (null-safety + error-handling often share one).
3. Cap at `--max-bundles` (default 6). When exceeded, merge the smallest bundles first.
4. Name each bundle with a kebab-slug describing the theme (e.g. `fix-acl`, `fix-fail-open`, `fix-concurrency`, `fix-nullsafety-misc`, `fix-data-integrity`, `fix-resources-perf`).

Print the bundle plan as a one-liner per bundle:

```
📦 fix-acl            3 bugs  [authz]        tdd?
📦 fix-concurrency    5 bugs  [concurrency]  architectural?
```

The `tdd?`/`architectural?` marker is a *guess* used only to prepare the reviewer model; the planner confirms it in frontmatter (Step 4b).

### Step 4b: Plan (1 opus planner per bundle)

Dispatch all planners in a single message:
- `isolation: "worktree"` — planners only **read**, but worktree isolation keeps their read state consistent
- `model: "opus"` (or `$MODEL_PLAN`)
- `name: "planner-<bundle>"`
- `run_in_background: true`

Each planner prompt includes:
- The bundle name and every bug claim JSON in it
- The recon report
- Project rules: `CLAUDE.md` + `AGENTS.md` content passed inline
- Instructions to:
  - Read each referenced file; understand shared root causes vs. independent issues
  - Produce a markdown plan at `.temp/bounty/plans/<bundle>.md` with YAML frontmatter:
    ```
    ---
    bundle: fix-acl
    test_strategy: tdd          # tdd | architectural — picks reviewer model (haiku vs. sonnet)
    commit_order: [BUG-authz-001, BUG-authz-002, BUG-authz-004]
    ---
    ```
  - One section per bug: proposed minimal fix, test stance (failing test that gets added, or why TDD won't work), and any dependencies on other bugs in the bundle
  - **No code changes.** Planning only. Read-only worktree contract: if the planner edits any file, its plan is rejected and the bundle falls back to per-bug fixing.

Poll `.temp/bounty/plans/` for new files between dispatch and the final `TaskList` wait; print a plan-written one-liner when each lands:

```
📋 plans/fix-acl.md written  test_strategy=tdd  3 bugs  reviewer=haiku
```

### Step 4c: Implement (1 sonnet implementer per bundle)

After all plans are written, dispatch implementers in a single message:
- `isolation: "worktree"` — one worktree per bundle, branch `bounty/<bundle>` off `main`
- `model: "sonnet"` (or `$MODEL_FIX`)
- `name: "implementer-<bundle>"`
- `run_in_background: true`

Each implementer prompt includes:
- The bundle's plan (`.temp/bounty/plans/<bundle>.md`) — passed verbatim
- The bundle's bug claims
- The recon report
- Project validation commands (from `CLAUDE.md` / `AGENTS.md`)
- Instructions to:
  - Walk the plan bug-by-bug in the order declared by `commit_order`
  - **TDD cycle per bug** when `test_strategy: tdd`: write the failing test first, confirm it fails, implement the minimal fix, confirm it passes, run validation, commit
  - One commit per bug. Never squash. Commit message follows the repo's `<module>: <benefit>` format; reference `BUG-<id>` in the body (never the title)
  - Run project validation after each bug. If a bug cannot pass, skip it and write `.temp/bounty/fixes/<BUG-ID>.json` with `{"status": "skipped", "reason": "..."}` — do not block the rest of the bundle
  - Write one `.temp/bounty/fixes/<BUG-ID>.json` per bug on completion (files changed, test added, validation output, commit SHA)
  - Return a final bundle summary: `<bundle>: <done>/<total> fixed, <skipped> skipped`

Tail `.temp/bounty/fixes/` on the same ~30s cycle and print the fix-committed one-liner per new report.

### Step 4d: Bundle review (1 reviewer per bundle)

After an implementer finishes its bundle, dispatch **1 reviewer**:
- `model: "haiku"` when the plan's frontmatter says `test_strategy: tdd` (the passing tests are most of the evidence)
- `model: "sonnet"` when `test_strategy: architectural` (review needs judgment)
- `name: "reviewer-<bundle>"`
- `isolation: "worktree"` (checks out `bounty/<bundle>`)
- Reviewer may not be the planner or implementer of this bundle. Prefer a specialist whose lens is adjacent but not represented among the bundle's original finders.

The reviewer inspects each per-bug commit on the bundle branch and returns one verdict per bug: `APPROVE` or `REJECT` with a one-sentence reason. Path: `.temp/bounty/reviews/<bundle>.json`.

Resolution per bug:
- `APPROVE` → fix kept on the bundle branch, implementer gets **+1**
- `REJECT` → revert that single commit on the bundle branch (`git revert <sha>`); the bug returns to the queue for **one** isolated retry by a different agent (per-bug fix, opus), then abandoned if still rejected

Print the bundle-review one-liner per bug and re-render the leaderboard after each change.

## Phase 5: Ship

Merge each bundle branch that has at least one approved fix into a single aggregate `bounty/run-<YYYYMMDD-HHMM>` off `main`.

Per-bug commits must be preserved — merge each bundle with `--no-ff --no-squash`, or cherry-pick the approved commits from a bundle when some of its commits were reverted during Step 4d. Never squash; the individual commits are the audit trail.

Run project validation on the aggregate branch. If it fails, bisect to find the offending commit and drop just that commit (`git revert`, then re-validate). Report which commit was dropped and which bug went with it.

Push the branch and open a PR via `gh pr create`:
- **Title**: `multiple: discovered and resolved N bugs via bounty run` (adjust module prefix if all fixes are in one module)
- **Body**: see template in `references/scoring-rules.md`

For each `critical` bug, open a separate GitHub issue instead of committing.

## Phase 6: Final report

Render the final leaderboard to stdout and to `.temp/bounty/README.md`:

```
🏆 FINAL STANDINGS
┌──────────────────┬───────┬──────┬─────┬───────┬───────┐
│ Specialist       │ Found │ Conf │ FPs │ Fixes │ SCORE │
├──────────────────┼───────┼──────┼─────┼───────┼───────┤
│ 🛡️  Security     │     5 │    4 │   1 │     2 │    +3 │
│ 🧵 Concurrency   │     3 │    3 │   0 │     1 │    +4 │
│ ...              │       │      │     │       │       │
└──────────────────┴───────┴──────┴─────┴───────┴───────┘
```

Report:
- PR URL (if any)
- List of critical issues opened
- Abandoned fixes (if any), with reason
- Total wall clock and rough token cost estimate

## Cleanup

The `/bounty-cleanup` command tears down worktrees and removes `bounty/*` branches that were merged.

## Rules — non-negotiable

1. Agents cannot vote on their own claims. The orchestrator enforces this by excluding the finder from that claim's voter set.
2. Agents cannot review fixes they wrote or claims they found.
3. Every claim needs a file path, line number, and evidence snippet. Claims missing any of these are rejected by the orchestrator before voting.
4. Follow the repo's commit rules (no AI attribution, no `fix:` / `feat:` prefixes, lowercase module prefix before colon).
5. All state lives under `.temp/bounty/`. Never mutate repo files outside fix worktrees.

## Additional Resources

- `references/agent-specialties.md` — the 8 specialist lenses with concrete patterns
- `references/scoring-rules.md` — scoring, thresholds, leaderboard and PR templates
- `references/state-schema.md` — JSON schemas for claims, votes, leaderboard, fixes
