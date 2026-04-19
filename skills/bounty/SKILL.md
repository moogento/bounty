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
| `--agents N` | 8 | Number of specialist agents (min 3, max 12) |
| `--severity <floor>` | `medium` | Discard claims below this severity |
| `--max-claims N` | 40 | Cap total claims to keep cost bounded |
| `--no-fix` | false | Skip Phase 4–6, just report findings |
| `--model-hunt <tier>` | `sonnet` | Model tier for hunters |
| `--model-fix <tier>` | `opus` | Model tier for fixers |

If no scope is given, ask the user once whether to scope the hunt.

## Phase 0: Initialize state

Create the state directory and seed files:

```bash
mkdir -p .temp/bounty/{claims,votes,fixes}
```

Write `.temp/bounty/config.json` with the resolved arguments.

Write an empty `.temp/bounty/leaderboard.json` with zeroed scores for the chosen specialists. See `references/state-schema.md` for file schemas.

## Phase 1: Shared recon

Spawn **1 recon agent** in an isolated worktree (`isolation: "worktree"`, `model: "sonnet"`, `run_in_background: false`). Its job: produce a shared briefing that every hunter receives as cached context.

Prompt must include:
- The scope (or "repo root")
- Instructions to:
  - Map the major modules and their responsibilities
  - Identify external boundaries (HTTP, DB, filesystem, third-party APIs, user input)
  - Run `git log --since="90 days ago" --stat` on the scope and note high-churn files
  - List test coverage gaps (files with no tests in the scope)
  - Return a **markdown report** (≤2k words) with sections: Modules, Boundaries, Recent churn, Coverage gaps, Suspicion heatmap
- A **5-minute time budget**

Save the report to `.temp/bounty/recon.md`. This is the shared briefing — pass it verbatim to every downstream agent so the prompt cache reuses it.

## Phase 2: The hunt (parallel specialists)

Assign each of N agents one **specialist lens** from `references/agent-specialties.md`. Any agent can still claim any bug (that is the rule), but the lens is their starting point to reduce pure overlap.

**Dispatch all N agents in a single message**, each with:
- `isolation: "worktree"`
- `name: "hunter-<lens>"` (so they are addressable later)
- `model: "sonnet"` (or `$MODEL_HUNT`)
- `run_in_background: true`
- `subagent_type: "general-purpose"` (unless a specialist subagent fits — e.g. `moo-security-reviewer` for the security lens)

Each prompt must include:
- Their specialist lens with 3–5 concrete patterns to watch for (pulled from `references/agent-specialties.md`)
- The scope
- The recon report from `.temp/bounty/recon.md`
- The severity floor
- Instructions to:
  - Scan their lens first, then broaden
  - For each claim, verify the evidence by reading the code — do not guess
  - Discard claims below the severity floor
  - **Write each claim as a JSON file** to `.temp/bounty/claims/BUG-<agent>-<seq>.json` (see `references/state-schema.md`). Never emit claims only to stdout — the file is the source of truth.
  - Stop when they have 5 claims OR 10 minutes elapsed, whichever comes first
  - Return a final one-line summary: `<lens>: <n> claims submitted`

Wait for all N agents. Read the claims directory.

### Deduplicate

Compute a fingerprint for each claim: `sha256(normalized_file + ":" + category + ":" + line_bucket)` where `line_bucket = line // 10`. Claims with the same fingerprint are duplicates — keep the first-submitted, attribute the later ones as "co-discovered" (no bonus, no penalty).

### Enforce `--max-claims`

If total claims exceed the cap, keep the highest-severity claims and drop the rest. Log the drop count.

## Phase 3: Parallel voting

For each surviving claim, the **other N−1 agents** vote. Dispatch all voters for a given claim **in a single message** using `model: "haiku"` (voting is a cheap verdict, not deep analysis).

Each voter prompt includes:
- The claim JSON
- The file excerpt cited as evidence (read it, include ±10 lines)
- Instructions: return a single JSON line `{"verdict": "VALID"|"FALSE_POSITIVE"|"ABSTAIN", "reason": "<one sentence>"}` appended to `.temp/bounty/votes/<BUG-ID>.jsonl`

Process claims in batches of 5 to keep parallelism bounded.

### Resolution rules

Read the tallied votes and apply:

| Outcome | Condition | Points |
|---------|-----------|--------|
| CONFIRMED | majority VALID among non-abstaining voters | finder **+1** |
| FALSE_POSITIVE | ≥⌈(N−1)/3⌉ FALSE_POSITIVE votes **and** majority is FALSE_POSITIVE | finder **−3** |
| INCONCLUSIVE | neither | 0 |

Update `.temp/bounty/leaderboard.json` atomically. Update `.temp/bounty/README.md` with the current leaderboard (see `references/scoring-rules.md` for format).

## Phase 4: Fix queue (skip if `--no-fix`)

Confirmed bugs enter `.temp/bounty/queue.json` with their severity.

### Severity policy

| Severity | Action |
|----------|--------|
| low | Fix + commit |
| medium | Fix + commit |
| high | Fix + commit + flag for human review in PR body |
| critical | Do **not** auto-fix. Open a GitHub issue with full evidence via `gh issue create` |

### Fix dispatch

For each fixable bug, spawn a fixer in an isolated worktree:
- `isolation: "worktree"`
- `model: "opus"` (or `$MODEL_FIX`)
- `name: "fixer-<BUG-ID>"`
- The finding agent gets first right of refusal; assign them by name. If they decline (or to distribute load), assign any other agent.

Each fixer prompt includes:
- The claim JSON
- The recon report
- The file evidence
- Project rules: `CLAUDE.md` + `AGENTS.md` content passed inline
- Instructions to:
  - Implement the minimal correct fix
  - Write a regression test if feasible; if not, explain why in the fix report
  - Run the project's validation (phpunit, phpcs — see `CLAUDE.md` for commands)
  - Write the fix report to `.temp/bounty/fixes/<BUG-ID>.json` with: files changed, test added (yes/no), validation output
  - Commit on its worktree branch `bounty/<BUG-ID>-<kebab-slug>` with a commit message following the repo's `modulename: benefit summary` format

Dispatch fixers in batches of 3 in parallel (opus cost is non-trivial).

### Fix review

For each completed fix, dispatch **2 reviewers in parallel** (`model: "sonnet"`), excluding the finder and fixer. Each returns `APPROVE` or `REJECT` with a one-sentence reason.

- 2 approvals → fix is kept, fixer gets **+1**
- 1+ rejections → fix returned to queue; a different agent may try once more, then abandon

## Phase 5: Ship

Merge all approved fix worktrees into a single branch `bounty/run-<YYYYMMDD-HHMM>` off `main`.

For each fix, preserve its individual commit (cherry-pick from its worktree). Do not squash.

Run project validation on the aggregate branch. If it fails, bisect to find the offending fix and drop it (report which).

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
