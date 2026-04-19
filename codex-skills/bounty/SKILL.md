---
name: bounty
description: Use when the user wants to run a bounty, hunt for unknown bugs, or scan a codebase with multiple competing specialists in Codex.
metadata:
  short-description: Competitive multi-agent bug discovery
---

# Bounty

Run a competitive multi-agent bug hunt adapted for Codex.

This port keeps the same seven phases as the Claude Code plugin, but the orchestration changes in one important way:

- The main Codex agent owns `.temp/bounty/` and performs every persistent write.
- Subagents return structured results in their final messages; the orchestrator validates those results and then serializes claims, votes, fix reports, and leaderboard updates into the shared state directory.
- A request to "run bounty", "start a bounty hunt", or similar language counts as explicit permission to use parallel Codex subagents.

Reuse the shared reference material from the Claude plugin instead of duplicating policy:

- `../../skills/bounty/references/agent-specialties.md`
- `../../skills/bounty/references/scoring-rules.md`
- `../../skills/bounty/references/state-schema.md`

## Arguments

Parse the user request for these optional flags or settings:

| Flag | Default | Meaning |
|------|---------|---------|
| `--scope <path>` | repo root | Restrict the hunt to a directory or glob |
| `--agents N` | 8 | Number of specialist lenses in play, 3-12 |
| `--hunters-per-lens K` | 1 | Hunters dispatched per lens; ≥2 treats collisions as validation |
| `--severity <floor>` | `medium` | Discard claims below this severity |
| `--max-claims N` | 40 | Cap total claims so the run stays bounded |
| `--no-fix` | false | Skip fix/review/ship and stop at findings |
| `--voting-mode <mode>` | `auto` | `full` = N−1 voters per claim. `panel` = 1 voter per specialty reviews every non-own claim. `auto` = `full` when claims ≤ 20, else `panel` |
| `--max-fixes N` | 10 | If confirmed count exceeds, Phase 4 pauses for user to confirm/curate the bundle plan |
| `--max-bundles N` | 6 | Cap on Phase-4 bundles |
| `--model-hunt <model>` | `gpt-5.4-mini` | Preferred model for recon, hunting, and voting |
| `--model-plan <model>` | `gpt-5.4` | Preferred model for bundle planners |
| `--model-fix <model>` | `gpt-5.4-mini` | Preferred model for bundle implementers |

If no scope is given, ask once whether to hunt the whole repo or a narrower module.

## Runtime Rules

1. The orchestrator creates and updates every file under `.temp/bounty/`.
2. Subagents do not write shared state directly. They return structured text that the orchestrator validates first.
3. Agents cannot vote on their own claims.
4. Reviewers cannot review fixes they wrote or findings they originally submitted.
5. Every claim needs a file path, line number, evidence snippet, trigger, impact, category, and severity. Missing any required field means the claim is rejected before voting.
6. Keep the run resumable. If state already exists, reuse it unless the user explicitly asks for a fresh run.

## Phase 0: Initialize state

Create the state directory:

```bash
mkdir -p .temp/bounty/{claims,votes,fixes,plans,reviews}
```

**Resolve the absolute state directory path** and store it as `state_dir` in `.temp/bounty/config.json`. Codex worker subagents can run in sandboxed directories — pass `STATE_DIR=<absolute>` in every dispatched prompt so outputs are serialized back to the main repo's state dir, never into a sandbox.

Write `.temp/bounty/config.json` using the schema from `../../skills/bounty/references/state-schema.md`, with the resolved arguments plus `"runtime": "codex"`.

Write `.temp/bounty/leaderboard.json` with zeroed scores for the chosen specialists. Use temp-file plus rename for atomic writes.

**Probe the voter-tier model** before the hunt: fire one throwaway call at `$MODEL_VOTE` and catch a rate-limit / usage-cap error. If it fires, warn the user:

```
⚠ voter model returned a rate-limit on probe; Phase 3 will fall back to the hunt-tier model for voting.
```

## Live progress output

The orchestrator must surface progress as events happen, not just at phase boundaries. Silent waits feel broken.

Because Codex subagents do not stream writes into the shared `.temp/bounty/` directory while they are still running, the live boundary is different from Claude Code:

- Treat each completed subagent result as an event boundary.
- Serialize accepted claims, votes, and fix reports into `.temp/bounty/` immediately when that subagent finishes.
- Print the event line immediately after validation and persistence. Do not wait for the rest of the phase to finish.

Rules that apply to every phase below:

1. Wait on background subagents incrementally, not as one big blocking batch. Use `wait_agent` on the outstanding agent ids with a short timeout, process whichever agent finishes first, then continue with the remaining set.
2. Print a one-liner per event as it lands. Use compact formats:
   - New claim: `🛡️ Security/1 → BUG-003  high  sql-injection  app/Http/Orders.php:142`
   - Collision: `🛡️ Security/2 collided with 🛡️ Security/1 on BUG-003  (collision_count = 2)`
   - New vote: `BUG-003  ✓ VALID  from 🧵 Concurrency  (sanitizer bypassed upstream)`
   - Claim resolved: `BUG-003 CONFIRMED (4 VALID / 1 FP / 2 abstain)  finder +1 → 🛡️ Security`
   - Bundle planned: `📦 fix-acl  3 bugs  test_strategy=tdd  reviewer=haiku`
   - Plan written: `📋 plans/fix-acl.md written  3 bugs  commit_order=[…]`
   - Fix committed: `BUG-003 (fix-acl) committed  (2 files, test added, phpunit ✓)`
   - Bundle reviewed: `fix-acl  3/3 APPROVE from 🧵  → kept, implementer +3`
3. Re-render the running leaderboard on every scoring change: new confirmation, false positive, or fix approval. Prefix it with `— leaderboard @ <phase> —`.
4. Never compress multiple new events into a single summary line. If three claims are accepted from one hunter result, print three claim lines.
5. If no agent completes for roughly 20-30 seconds, emit a short heartbeat update that names the phase and the remaining agent count, then keep waiting.
6. Refresh `.temp/bounty/README.md` whenever the visible leaderboard changes so the on-disk dashboard matches what the user sees.

## Phase 1: Shared recon

Spawn one bounded recon subagent. Prefer:

- `agent_type: "explorer"`
- model `gpt-5.4-mini` or the user-provided hunt model

Its job:

- map major modules and responsibilities inside scope
- identify external boundaries: HTTP, DB, filesystem, queues, third-party APIs, user input
- inspect recent churn, for example `git log --since="90 days ago" --stat -- <scope>`
- list obvious coverage gaps
- produce a markdown report with sections: Modules, Boundaries, Recent churn, Coverage gaps, Suspicion heatmap

**Scale the time budget by scope size**: `budget_minutes = max(5, ceil(files_in_scope / 100))`. Record the chosen budget as `recon_budget_minutes` in `config.json`. Save the returned report to `$STATE_DIR/recon.md`.

## Phase 2: The hunt

Assign each of `N` hunters one specialist lens from `../../skills/bounty/references/agent-specialties.md`.

When `--hunters-per-lens K > 1`, dispatch **K hunters per lens** as a single flat parallel wave (total = N×K agents). Each hunter in a lens receives a distinct **prompt seed** so they take different paths through the code:

- seed 1: breadth-first from the top of the module tree
- seed 2: follow high-churn files from the recon report
- seed 3: follow coverage gaps from the recon report

Collisions between hunters on the same lens are the validation signal. Record them as Phase 2 proceeds.

Dispatch the hunters in parallel as read-heavy subagents. Prefer:

- `agent_type: "explorer"`
- model `gpt-5.4-mini` or the configured hunt model (all seeds on the same tier; second/third hunters exist to cross-check cheaply, not to deepen)

Each hunter receives:

- their specialist lens and concrete patterns
- the prompt seed (if K>1) — as a starting point, not a cage
- the resolved scope
- the recon report
- the severity floor
- the claim schema from `state-schema.md`

Each hunter must:

- scan their lens from the seed entry point first, then broaden
- verify evidence from code before making a claim
- **respect a tool-call budget** — cap read-style calls at ~20 per hunter; if they need more, the scope is too wide, submit what they have
- **hard stop** the moment 5 claims are filed; do not keep exploring past that
- return at most 5 claims as JSON objects in the final message
- stop after 5 claims or roughly 10 minutes of effort

### Live progress while hunters run

Do not dispatch all hunters and then block until all of them finish.

Instead:

1. Keep a set of outstanding hunter agent ids.
2. Call `wait_agent` on that set with a bounded timeout so you get whichever hunter finishes first.
3. As soon as a hunter completes, validate each returned claim, write its JSON file immediately, and print the new-claim one-liner for each accepted claim.
4. After processing that hunter, print a compact running count such as `claims so far: 🛡️ 3 · 🧵 2 · ⚡ 1 · … (total 6)`.
5. Continue waiting on the remaining hunters until the set is empty.

The orchestrator then:

- validates every returned claim
- computes the fingerprint `sha256(normalized_file + ":" + category + ":" + (line // 10))`
- writes accepted claims to `.temp/bounty/claims/BUG-<lens>-<seed>-<seq>.json`
- merges duplicates by fingerprint and records later submissions in `co_discovered_by`
- increments `collision_count` on the surviving claim each time a duplicate lands; emits a `collided` one-liner live
- runs a **secondary semantic-dedup pass** over surviving claims: buckets by `(normalize(file), line // 10)` ignoring category, and for any bucket with ≥2 claims from different specialties, merges them (keep highest-severity as survivor, append others' finders to `co_discovered_by`, increment `collision_count`, record merged categories). Emits a semantic-merge one-liner, e.g. `🔗 semantic merge: BUG-authz-1-001 + BUG-security-1-004 → surviving BUG-security-1-004`
- enforces `--max-claims` by keeping the highest-severity claims first

## Phase 3: Voting

Resolve the voting mode from `--voting-mode` first. Announce it at phase start:

- `full` — for each claim, dispatch `N−1` voter subagents (independent verdicts per claim). Use when `surviving_claims ≤ 20`.
- `panel` — dispatch one voter subagent per specialty; each reviews every claim not authored by its own lens and returns verdicts for all of them in a single structured return. The orchestrator serializes each voter's return into `$STATE_DIR/votes/VOTER-<lens>.jsonl` (one line per verdict). Use when `surviving_claims > 20`.
- `auto` (default) — resolve at runtime based on claim count.

If the voter-tier model rate-limits, announce once and fall back to `$MODEL_HUNT` for voting; persist the fallback in `config.json` as `model_vote_effective`.

### Full-mode voting

For each surviving claim, collect verdicts from the other `N-1` specialists. Process claims in batches of 5 so the run stays bounded.

Each voter receives:

- the claim JSON
- the cited file excerpt with about 10 lines of context on each side
- instructions to return exactly one JSON object:

```json
{"verdict":"VALID|FALSE_POSITIVE|ABSTAIN","reason":"one sentence"}
```

Prefer smaller, cheaper subagents for voting:

- `agent_type: "explorer"`
- model `gpt-5.4-mini`

### Live progress during voting

Do not wait for a whole claim batch to finish before showing progress.

Instead:

1. Track outstanding voter agent ids for the active batch.
2. Use `wait_agent` with a short timeout and process each completed voter as soon as it returns.
3. Append that vote immediately to `.temp/bounty/votes/<BUG-ID>.jsonl` and print the new-vote one-liner.
4. As soon as all votes for a given claim are present, resolve that claim immediately, update the leaderboard, print the claim-resolved one-liner, and re-render the leaderboard table.
5. Continue until the batch has no unresolved claims left, then move to the next batch.

The orchestrator appends every vote to `.temp/bounty/votes/<BUG-ID>.jsonl`, tallies the result, and updates `.temp/bounty/leaderboard.json` using the thresholds from `../../skills/bounty/references/scoring-rules.md`.

Do not wait for the end of the batch to update the visible standings. The leaderboard update happens claim-by-claim.

## Phase 4: Bundled fixes

Skip this phase when `--no-fix` is set.

Instead of one fixer per bug, Phase 4 runs a **plan → implement → review** pipeline over bundles. For 30+ confirmed bugs this is the difference between ~90 model calls and ~18.

### Volume gate

Before dispatching planners, check `confirmed_count` against `--max-fixes` (default 10). If exceeded, ping the user with the bundle plan and **wait up to 3 minutes** — do not block indefinitely. Options: `all` / `high-only` / `pick <slugs>` / `abort`. On reply, honor the choice and record `user_fix_scope` in `config.json`. On timeout, record `user_fix_scope: "all"` with `gate_timeout: true`, announce the timeout, and continue with every bundle. Skip the gate entirely when `confirmed_count ≤ max_fixes`.

Severity policy:

- `low`, `medium`, `high`: eligible for bundled fixes
- `critical`: do not auto-fix; open a GitHub issue if GitHub access is available, otherwise record the blocker clearly in the final report

### Step 4a: Bundle confirmed bugs

Cluster fixable bugs and write `.temp/bounty/bundles.json`. Default heuristic:

1. One bundle per lens with ≥2 confirmed bugs.
2. Lone-bug lenses merge into a `misc` bundle alongside adjacent lenses.
3. Cap at `--max-bundles` (default 6); merge smallest first when exceeded.
4. Each bundle has a kebab-slug name (e.g. `fix-acl`, `fix-fail-open`, `fix-concurrency`, `fix-nullsafety-misc`).

### Step 4b: Plan

Dispatch **one planner subagent per bundle** in parallel:

- `agent_type: "worker"`
- model `gpt-5.4` or the configured plan model

Each planner gets the bundle name, every bug claim in the bundle, the recon report, and repo contribution rules.

Each planner must return a markdown plan with YAML frontmatter:

```
---
bundle: fix-acl
test_strategy: tdd          # tdd | architectural — picks reviewer model
commit_order: [BUG-authz-1-001, BUG-authz-1-002]
---

# <bundle> plan
## BUG-authz-1-001 — …
- Proposed fix: …
- Test stance: …
- Dependencies: …
```

Planners **must not produce code edits**. The orchestrator validates that the returned plan has no patch/diff content; if it does, the plan is rejected and that bundle falls back to per-bug fixing.

The orchestrator writes the plan to `.temp/bounty/plans/<bundle>.md` and prints a plan-written one-liner.

Process planners incrementally via `wait_agent` so each plan becomes available as soon as it's ready.

### Step 4c: Implement

After all plans are written, dispatch **one implementer subagent per bundle**:

- `agent_type: "worker"`
- model `gpt-5.4-mini` or the configured fix model
- each gets its bundle's plan (verbatim), the claim JSONs, the recon report, and project validation commands

Each implementer must walk `commit_order` one bug at a time:

- when `test_strategy: tdd`, write the failing test first, then the fix
- produce a minimal patch or a precise edit plan per bug; one commit per bug, never squashed
- commit message follows the repo `<module>: <benefit>` format; reference the BUG-ID in the body
- if a bug cannot pass validation, mark it `status: "skipped"` with a reason and continue the bundle
- return a per-bug report the orchestrator serializes to `.temp/bounty/fixes/<BUG-ID>.json`

The orchestrator owns integration:

- apply the returned patch/edits in the bundle's worktree on branch `bounty/<bundle>`
- run project validation after each applied bug
- write `.temp/bounty/fixes/<BUG-ID>.json` and print the fix-committed one-liner
- commit on the bundle branch only when the per-bug validation passes

## Phase 5: Bundle review and ship

For each completed bundle, dispatch **one reviewer subagent** — not the planner or implementer of that bundle, and not a bug's original finder when avoidable:

- `haiku`-tier model when the bundle's plan frontmatter says `test_strategy: tdd`
- `sonnet`-tier (`gpt-5.4`) when `test_strategy: architectural`

The reviewer gets the plan, the patches/diffs for every per-bug commit on the bundle branch, and the validation output. It returns one verdict per bug (`APPROVE` / `REJECT` with a one-sentence reason). The orchestrator writes `.temp/bounty/reviews/<bundle>.json`.

Process reviewers incrementally via `wait_agent`; the moment a bundle's review lands, resolve it and re-render the leaderboard.

Resolution per bug:

- `APPROVE`: keep the commit; implementer gets `+1`
- `REJECT`: revert that single commit on the bundle branch; the bug goes back to the queue for one isolated per-bug retry (opus/`gpt-5.4`), then abandoned if it rejects again

After review:

- create an aggregate branch `bounty/run-<YYYYMMDD-HHMM>`
- merge each bundle branch that retained at least one commit (`--no-ff --no-squash`) — per-bug commits are the audit trail, never squashed
- run aggregate validation
- if aggregate validation fails, bisect and drop the offending commit before creating the PR

Use `gh` or the GitHub connector when available to create:

- one PR for the approved non-critical fixes
- separate issues for critical bugs

If GitHub tooling is unavailable, stop after local commits and report the exact blocker.

## Phase 6: Final report

Render final standings to stdout and `.temp/bounty/README.md`, including:

- confirmed bugs
- false positives
- fixes approved
- abandoned fixes and reasons
- PR URL if created
- critical issues opened
- rough wall-clock and cost notes

## Notes

- Discovery should stay parallel; integration can become serial if that is the safer way to preserve correctness.
- Do not silently skip phases because Codex and Claude have different subagent mechanics. Adapt the orchestration, not the workflow.
- The Codex port should feel live even though the mechanics differ: process the first finished subagent immediately, not the whole wave at once.
