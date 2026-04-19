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
| `--agents N` | 8 | Number of specialist agents, 3-12 |
| `--severity <floor>` | `medium` | Discard claims below this severity |
| `--max-claims N` | 40 | Cap total claims so the run stays bounded |
| `--no-fix` | false | Skip fix/review/ship and stop at findings |
| `--model-hunt <model>` | `gpt-5.4-mini` | Preferred model for recon, hunting, and voting |
| `--model-fix <model>` | `gpt-5.4` | Preferred model for fix proposals |

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
mkdir -p .temp/bounty/{claims,votes,fixes}
```

Write `.temp/bounty/config.json` using the schema from `../../skills/bounty/references/state-schema.md`, with the resolved arguments plus `"runtime": "codex"`.

Write `.temp/bounty/leaderboard.json` with zeroed scores for the chosen specialists. Use temp-file plus rename for atomic writes.

## Live progress output

The orchestrator must surface progress as events happen, not just at phase boundaries. Silent waits feel broken.

Because Codex subagents do not stream writes into the shared `.temp/bounty/` directory while they are still running, the live boundary is different from Claude Code:

- Treat each completed subagent result as an event boundary.
- Serialize accepted claims, votes, and fix reports into `.temp/bounty/` immediately when that subagent finishes.
- Print the event line immediately after validation and persistence. Do not wait for the rest of the phase to finish.

Rules that apply to every phase below:

1. Wait on background subagents incrementally, not as one big blocking batch. Use `wait_agent` on the outstanding agent ids with a short timeout, process whichever agent finishes first, then continue with the remaining set.
2. Print a one-liner per event as it lands. Use compact formats:
   - New claim: `🛡️ Security → BUG-003  high  sql-injection  app/Http/Orders.php:142`
   - New vote: `BUG-003  ✓ VALID  from 🧵 Concurrency  (sanitizer bypassed upstream)`
   - Claim resolved: `BUG-003 CONFIRMED (4 VALID / 1 FP / 2 abstain)  finder +1 → 🛡️ Security`
   - Fix completed: `BUG-003 fixed by 🛡️ Security  (2 files, test added, phpunit ✓)`
   - Fix review: `BUG-003  APPROVE from 🧵  APPROVE from ⚡  → kept, fixer +1`
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

Time-box recon to roughly five minutes of effort. Save the returned report to `.temp/bounty/recon.md`.

## Phase 2: The hunt

Assign each of `N` hunters one specialist lens from `../../skills/bounty/references/agent-specialties.md`.

Dispatch the hunters in parallel as read-heavy subagents. Prefer:

- `agent_type: "explorer"`
- model `gpt-5.4-mini` or the configured hunt model

Each hunter receives:

- their specialist lens and concrete patterns
- the resolved scope
- the recon report
- the severity floor
- the claim schema from `state-schema.md`

Each hunter must:

- scan their lens first, then broaden if needed
- verify evidence from code before making a claim
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
- writes accepted claims to `.temp/bounty/claims/BUG-<lens>-<seq>.json`
- merges duplicates by fingerprint and records later submissions in `co_discovered_by`
- enforces `--max-claims` by keeping the highest-severity claims first

## Phase 3: Voting

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

## Phase 4: Fix queue

Skip this phase when `--no-fix` is set.

Build `.temp/bounty/queue.json` from confirmed bugs and apply the severity policy from the shared references:

- `low`, `medium`, `high`: eligible for fixes
- `critical`: do not auto-fix; open a GitHub issue if GitHub access is available, otherwise record the blocker clearly in the final report

For fixable bugs, prefer up to 3 fixer subagents in parallel:

- `agent_type: "worker"`
- model `gpt-5.4` or the configured fix model

Each fixer gets:

- the claim JSON
- the recon report
- the relevant code excerpt
- repo-specific contribution rules if present, such as `CLAUDE.md`, `AGENTS.md`, or other local guidance files

Each fixer must return:

- a minimal patch or a precise edit plan the orchestrator can apply
- whether a regression test was added
- validation notes and commands run

### Live progress during fixes

Treat each completed fixer as its own live event.

1. Keep a set of outstanding fixer agent ids for the current batch.
2. Use `wait_agent` incrementally so you process whichever fixer finishes first.
3. As soon as a fixer returns something usable, integrate it, write `.temp/bounty/fixes/<BUG-ID>.json`, and print the fix-completed one-liner.
4. When a fix later clears review, re-render the leaderboard immediately rather than waiting for the end of all fixes.

The orchestrator owns integration:

- apply or recreate the fix locally
- run the relevant validation
- write `.temp/bounty/fixes/<BUG-ID>.json`
- commit on `bounty/<BUG-ID>-<kebab-slug>` when the fix is accepted

## Phase 5: Fix review and ship

For every completed fix, collect two reviewer verdicts from agents other than the finder and fixer.

Reviewers inspect:

- the claim JSON
- the proposed patch or diff
- the validation output

Process reviewer completions incrementally as well. When the second reviewer for a fix completes, resolve that fix immediately, print the fix-review one-liner, and update the leaderboard if the fix is kept.

Resolution:

- `2/2 APPROVE`: keep the fix and award the fixer `+1`
- any rejection: give one different fixer one more attempt, then abandon if it still does not pass review

After approvals:

- create an aggregate branch `bounty/run-<YYYYMMDD-HHMM>`
- cherry-pick approved fix commits onto it
- run aggregate validation
- if aggregate validation fails, identify and drop the offending fix before creating the PR

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
