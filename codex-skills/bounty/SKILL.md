---
name: bounty
description: Use when the user wants to run a bounty, hunt for unknown bugs, or scan a codebase with multiple competing specialists in Codex.
metadata:
  short-description: Competitive multi-agent bug discovery
---

# Bounty

Run a competitive multi-agent bug hunt adapted for Codex.

This port keeps the same seven phases as the Claude Code plugin, but the orchestration changes in one important way:

- The main Codex agent owns `$STATE_DIR` (= `.temp/bounty/<run_id>/` for this run) and performs every persistent write.
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
| `--poc <mode>` | `auto` | Phase 3.5 PoC gate. `auto` = run for every CONFIRMED claim with severity ≥ `high`. `always` = every CONFIRMED claim. `off` = skip. A PoC that fails to reproduce demotes the claim to FALSE_POSITIVE (finder −3) |
| `--secrets-scan` / `--no-secrets-scan` | on | Step 5b.5 scans each aggregate branch's diff for credential patterns before `gh pr create`. A hit blocks that PR group only; other groups still ship |
| `--no-fix` | false | Skip fix/review/ship and stop at findings |
| `--voting-mode <mode>` | `auto` | `full` = N−1 voters per claim. `panel` = 1 voter per specialty reviews every non-own claim. `auto` = `full` when claims ≤ 20, else `panel` |
| `--max-fixes N` | 10 | If confirmed count exceeds, Phase 4 pauses for user to confirm/curate the bundle plan |
| `--max-bundles N` | 6 | Cap on Phase-4 bundles |
| `--model-hunt <model>` | `gpt-5.4-mini` | Preferred model for recon and hunting |
| `--model-vote <model>` | `gpt-5.4-mini` | Preferred model for voting; falls back to `--model-hunt` if it rate-limits |
| `--model-plan <model>` | `gpt-5.4` | Preferred model for bundle planners |
| `--model-fix <model>` | `gpt-5.4-mini` | Preferred model for bundle implementers |
| `--max-bugs-per-pr N` | 10 | Phase-5 splits into an additional PR when a group would exceed this bug count |
| `--max-lines-per-pr N` | 400 | Phase-5 splits into an additional PR when a group's added+removed line count would exceed this |
| `--pr-review-rounds N` | 2 | Step 5e polls each opened PR for automated-review comments, validates them, fixes valid ones, and (when replies are on) replies with rationale to invalid ones, up to N rounds per PR |
| `--pr-review-wait <secs>` | 300 | How long to wait after opening a PR before the first round of feedback polling (gives CI + bot reviewers time to post) |
| `--pr-review-replies` / `--no-pr-review-replies` | ask | Whether Step 5e posts a rationale reply on each PR for suggestions marked `INVALID`. Explicit flag skips the Phase-0 verify; when neither is set, verify with the user at Phase 0 (default `on` after 30s timeout). Resolved choice is recorded in `config.json` as `pr_review_replies` |
| `--dismissed-file <path>` | `.bounty/dismissed.yml` | Per-repo memory of patterns previously confirmed NOT to be bugs. Loaded at Phase 0d, passed to hunters, used to auto-dismiss matching claims before voting |
| `--persist-dismissed` / `--no-persist-dismissed` | ask | Whether to offer appending voter-confirmed false positives to the dismissed file at end of Phase 3. Explicit flag skips the prompt; neither flag → ask the user when at least one FP lands (default `on` after 30s timeout) |
| `--ignore-dismissed` | false | Debugging/rediscovery: skip the dismissed-file load entirely this run |
| `--fresh` | false | At Phase 0, proceed alongside any existing `.temp/bounty/<id>/` dirs without prompting (each run uses its own state dir) |
| `--resume <id>` | — | At Phase 0, reuse an existing `.temp/bounty/<id>/` state dir and continue from its last completed phase. `--resume` with no id resumes the sole in-progress run when there is exactly one |

If no scope is given, ask once whether to hunt the whole repo or a narrower module.

## Runtime Rules

1. The orchestrator creates and updates every file under `$STATE_DIR`. Each run has its own per-run state directory (`.temp/bounty/<run_id>/`) so concurrent bounty runs never collide.
2. Subagents do not write shared state directly. They return structured text that the orchestrator validates first.
3. Agents cannot vote on their own claims.
4. Reviewers cannot review fixes they wrote or findings they originally submitted.
5. Every claim needs a file path, line number, evidence snippet, trigger, impact, category, and severity. Missing any required field means the claim is rejected before voting.
6. Keep the run resumable. If state already exists, reuse it unless the user explicitly asks for a fresh run.
7. Every branch this run creates is `bounty/<run_id>/<bundle>` (or `bounty/<run_id>/pr-*`). Never commit to `main` or the user's starting branch. The orchestrator enforces this via the Branch isolation contract and runs a leak sweep after every worker phase.
8. `bundles.json` must only reference `bug_id`s that exist in `$STATE_DIR/claims/`. Any desync aborts Phase 4 with a clear recovery message.
9. **Never fall back to the main-repo checkout when a worktree worker is blocked.** A blocked worker yields a `status: "blocked"` fix report, not a shortcut that writes to main. See the "fall back to main" anti-pattern in the Branch isolation contract.

## Phase 0: Initialize state

Phase 0 is five gates — run them in order. Past runs skipped gates and produced state pollution, branch collisions, and commits on the wrong branch.

### 0a. Generate the run id

```bash
SCOPE_SLUG=$(echo "${SCOPE:-repo}" | tr '/' '-' | tr -cd '[:alnum:]-' | sed 's/^-*//;s/-*$//' | cut -c1-24)
RUN_ID="$(date -u +%Y%m%d-%H%M%S)-${SCOPE_SLUG}"
```

Record `run_id` in `config.json`. Every branch MUST be named `bounty/<run_id>/<bundle>` (see Branch isolation contract). Two concurrent bounty runs in the same repo scope to different run ids and therefore different branches.

### 0b. Scan for existing runs

Each run's state lives under its own per-run directory: `.temp/bounty/<run_id>/`. Concurrent bounty runs in the same repo are supported — they never share a state dir.

Scan for any in-progress runs (`.temp/bounty/<id>/config.json` with `last_completed_phase < 6`):

| Condition | Action |
|-----------|--------|
| `--resume <id>` given | Load that run's config, adopt its `run_id`, continue from the first incomplete phase |
| `--resume` with no id, exactly one in-progress | Resume that one |
| `--fresh` given | Leave existing runs alone; create this run's own `.temp/bounty/<run_id>/` |
| No flag, an in-progress run overlaps this run's `--scope` | Surface it (`<id> · last-completed=<phase> · scope=<path>`). Ask: `resume-<id> / proceed-alongside / abort`. Default `proceed-alongside` on 30-second timeout |
| Nothing in-progress | Proceed |

"Overlap" means one scope is a prefix of the other. Non-overlapping scopes run alongside each other silently.

Never write into another run's state directory. The per-run layout makes this physically impossible as long as every write goes through `$STATE_DIR`.

### 0c. Working-tree and branch gate

```bash
MAIN_REPO="$(git rev-parse --show-toplevel)"
MAIN_START_BRANCH="$(git -C "$MAIN_REPO" branch --show-current)"
MAIN_START_HEAD="$(git -C "$MAIN_REPO" rev-parse HEAD)"
MAIN_START_DIRTY="$(git -C "$MAIN_REPO" status --porcelain | grep -vE '^\?\? \.temp/bounty' || true)"
```

Record all three in `config.json` under `main_start`. Block and prompt when any of the following is true:

1. `MAIN_START_BRANCH` is not the repo's default branch — commits from this run must land on fresh `bounty/<run_id>/*` branches cut from `origin/main`, not on top of the user's work.
2. `MAIN_START_DIRTY` is non-empty and any dirty path falls under `--scope` — this is the overlap that has caused past runs to stop mid-phase.

Ask: `stash-and-proceed / commit-first / proceed-anyway / abort`. On `stash-and-proceed`, `git stash push --include-untracked -m "bounty-<run_id>: pre-run stash"` and record the stash ref so `/bounty-cleanup` can surface it later.

### 0d. Initialize state directory

State is per-run, under `.temp/bounty/<run_id>/`:

```bash
STATE_DIR="$MAIN_REPO/.temp/bounty/$RUN_ID"
mkdir -p "$STATE_DIR"/{claims,votes,fixes,plans,reviews,pr-review}
ln -snf "$RUN_ID" "$MAIN_REPO/.temp/bounty/latest"
```

Store the absolute `state_dir` in `$STATE_DIR/config.json`. Codex worker subagents can run in sandboxed directories — pass `STATE_DIR=<absolute>` and `RUN_ID=<run_id>` in every dispatched prompt so outputs are serialized back to the main repo's state dir, never into a sandbox, and never into a *different* run's state dir if another bounty is active.

Write `$STATE_DIR/config.json` using the schema from `../../skills/bounty/references/state-schema.md`, with the resolved arguments plus `"runtime": "codex"`, `run_id`, and `main_start`.

**Seed `leaderboard.json` with every specialist in the `specialists` list, not just the ones we expect to find bugs.** A recent Claude-side run initialized only 5 specialists and silently dropped the other 3 when they submitted claims. Iterate the resolved specialist list; write a zeroed row for each one. Use temp-file plus rename for atomic writes.

**Write `leaderboard.json` (and every subsequent update) as single-line minified JSON via shell, not through a Write-style tool.** In Claude Code, the `Write` tool renders a multi-line JSON preview that visually dominates the frame and buries the championship table the user is meant to see. Codex should match: `jq -cn ... > tmp && mv tmp leaderboard.json`. The JSON file is machine state; the ASCII table in 0e is the user-facing artifact. Keep the file-write quiet so the table can be read.

**Resolve `pr_review_replies` before Phase 1.** Step 5e can post a consolidated rationale reply on each PR explaining why a suggestion was marked `INVALID`. Because this writes to a user-visible place, default to verifying — but only when the user didn't already express a preference on the command line:

| Command-line input | Resolved `pr_review_replies` | Prompt? |
|--------------------|------------------------------|---------|
| `--pr-review-replies` | `true` | No |
| `--no-pr-review-replies` | `false` | No |
| `--no-fix` (no PRs will exist) | `false` | No |
| neither flag, `--no-fix` absent | ask the user | Yes |

Prompt:

```
🤖 Step 5e can reply on each PR with a rationale for any suggestion it marks INVALID.
   Post these rationale replies?  (y/n)  — default y in 30s
```

On `y` / timeout: `pr_review_replies: true`. On `n`: `pr_review_replies: false`. Record the resolved value in `config.json` so `--resume` does not re-prompt. Announce the outcome once: `✓ PR rationale replies: on` or `… off`. Step 5e still records every INVALID verdict in `$STATE_DIR/pr-review/<pr-number>.json` regardless of the flag — only the GitHub-facing comment is gated.

**Load the per-repo dismissed-bug memory.** The repo can keep a list of patterns already determined NOT to be bugs under `--dismissed-file` (default `.bounty/dismissed.yml` at the repo root, outside `$STATE_DIR` so it survives across runs). Schema + matching rules: see `../../skills/bounty/references/state-schema.md#bountydismissedyml-per-repo-outside-state_dir`.

Load the file at Phase 0 (skip if missing or `--ignore-dismissed`). Snapshot into `$STATE_DIR/dismissed.yml` so the run is self-contained; all downstream reads use the snapshot. Announce once: `📒 dismissed list: <N> entries loaded from .bounty/dismissed.yml`. The list will be (1) inlined into every hunter prompt, (2) used at Phase 3 start to pre-filter claims as auto-FP, and (3) optionally appended to with voter-confirmed FPs at end of Phase 3.

Resolve `persist_dismissed` the same three-state way as `pr_review_replies`:

| Command-line | Resolved | Prompt? |
|--------------|----------|---------|
| `--persist-dismissed` | `true` | No |
| `--no-persist-dismissed` | `false` | No |
| `--ignore-dismissed` | `false` | No |
| neither | ask at end of Phase 3 | Yes |

Record the resolved value in `config.json`.

**Probe the voter-tier model** before the hunt: fire one throwaway call at `$MODEL_VOTE` and catch a rate-limit / usage-cap error. If it fires, warn the user:

```
⚠ voter model returned a rate-limit on probe; Phase 3 will fall back to the hunt-tier model for voting.
```

### 0e. Render the starting championship table — REQUIRED OUTPUT

**The single most-missed step in the entire skill.** Past runs have skipped straight to "dispatching recon…" and the user never sees a table until the final report (or never at all, if the run dies early). Do this now; it is not optional.

**A `leaderboard.json` file-write preview is not this table.** If the last thing in the user's view is a JSON blob, the scoreboard has not been rendered yet. Emit the table below as assistant stdout text (outside any tool call) immediately after writing `leaderboard.json`, so the table — not the JSON preview — is the most recent thing on screen.

Print this to stdout **and** write it to `$STATE_DIR/README.md` before any Phase-1 dispatch:

```
🏆 BOUNTY CHAMPIONSHIP — standings @ phase 0 (run <RUN_ID>)
┌─────┬──────────────────┬───────┬──────┬─────┬───────┬───────┐
│ #   │ Specialist       │ Found │ Conf │ FPs │ Fixes │ SCORE │
├─────┼──────────────────┼───────┼──────┼─────┼───────┼───────┤
│  -  │ 🛡️  Security     │     0 │    0 │   0 │     0 │     0 │
│  -  │ 🧵 Concurrency   │     0 │    0 │   0 │     0 │     0 │
│  -  │ ⚡ Performance   │     0 │    0 │   0 │     0 │     0 │
│  -  │ ∅ NullSafety     │     0 │    0 │   0 │     0 │     0 │
│  -  │ ⚠ ErrorHandling  │     0 │    0 │   0 │     0 │     0 │
│  -  │ 🔐 AuthZ         │     0 │    0 │   0 │     0 │     0 │
│  -  │ 🧮 DataIntegrity │     0 │    0 │   0 │     0 │     0 │
│  -  │ 💧 Resources     │     0 │    0 │   0 │     0 │     0 │
└─────┴──────────────────┴───────┴──────┴─────┴───────┴───────┘
 Confirmed: 0 │ FPs: 0 │ Inconclusive: 0 │ Fixed: 0 │ Pending: 0
```

Include exactly the specialists this run configured — no phantom rows, no dropped ones.

Re-render at every phase boundary (start of Phase 1, 2, 3, 4, 5, 6) with live numbers, prefixed `🏆 BOUNTY CHAMPIONSHIP — standings @ phase N (run <RUN_ID>)`. A long silent hunt phase is exactly when the user most needs to see the table, so the rule "re-render on scoring events" alone is not enough.

If the orchestrator has not emitted this exact table within the first 10 seconds of the run, it has violated the skill. Stop and emit it.

### 0f. Clawd the mascot (optional whimsy)

The run is a competition — let it feel like one. On every leaderboard re-render, print a short Clawd frame **above** the table. The frame varies with the most recent event so successive renders subtly animate. Single-frame per render only — multi-frame redraws flicker in a tailed terminal.

- New confirmation:
  ```
    /\___/\
   ( ^   ^ )   🎯  nice catch!
   (  =^=  )
    ‾‾‾‾‾‾
  ```
- False positive:
  ```
    /\___/\
   ( x   x )   😅  ope, false positive.
   (  =^=  )
    ‾‾‾‾‾‾
  ```
- Fix approved / PR landed:
  ```
    /\___/\    🏆
   ( ^   ^ )── 🏆  fix landed!
   (  =^=  )   🏆
    ‾‾‾‾‾‾
  ```
- Lead change:
  ```
     ♛ ♛ ♛
    /\___/\
   ( ^   ^ )   👑  <specialist> takes the crown!
   (  =^=  )
    ‾‾‾‾‾‾
  ```
- Quiet phase tick (no scoring change):
  ```
    /\___/\
   ( o   o )   🔎  still hunting…
   (  =^=  )
    ‾‾‾‾‾‾
  ```

Skip Clawd if memory records the user prefers terse output. The table is the contract; Clawd is garnish. When in doubt, render him.

## Branch isolation contract

All commits produced by a bounty run land on branches named `bounty/<run_id>/<bundle>` cut from `origin/main` inside isolated worktrees. **Never** on the user's current branch. **Never** on `main`.

### The "fall back to main" anti-pattern — forbidden

A prior run logged: *"worktree-isolated implementer agents got blocked on git/php -l permission prompts, so I completed the last fixes directly on the main-repo checkout."* **This must never happen again.** The orchestrator does not work around worktree permission prompts by moving work to the main repo. If a worktree worker is blocked on a tool prompt:

1. **Retry once** with the specific tool explicitly permitted in the dispatch (e.g. `Bash(php:*)`, `Bash(git:*)`). If it still blocks, proceed to step 2.
2. **Mark the bundle blocked, not complete.** Write `$STATE_DIR/fixes/<BUG-ID>.json` with `{"status": "blocked", "reason": "worktree tool prompt: <tool>"}` for the remaining bugs. Phase 5 skips blocked bundles.
3. **Surface to the user** with one line per blocked bundle: `⛔ fix-<bundle> blocked by worktree tool prompt for <tool>`.

Do **not** `cd` into the main repo, use the orchestrator's own permissions to apply the fix "just this once," or copy partial worker output to the main repo to finish it. Blocked fixes are a clean failure; leaks onto main are a catastrophic one.

### Post-implementer commit verification

After each implementer returns, before accepting its fix report:

```bash
for sha in $(jq -r '.commit_sha' "$STATE_DIR"/fixes/BUG-*.json); do
    ON_BUNDLE=$(git branch --contains "$sha" 2>/dev/null | grep -c "bounty/<run_id>/<bundle>" || echo 0)
    ON_MAIN=$(git branch --contains "$sha" 2>/dev/null | grep -c "^\s*main$" || echo 0)
    [ "$ON_BUNDLE" -eq 1 ] && [ "$ON_MAIN" -eq 0 ] || echo "LEAK $sha"
done
```

Any commit that lands on `main` (or `MAIN_START_BRANCH`) is a leak. Stop the run, recover the commit onto `bounty-recovered-<run_id>-<bundle>`, and restore main to `MAIN_START_HEAD`. Report.

### Worker prompt block

Every planner, implementer, and reviewer prompt must include this verbatim:

```
You are in an isolated worktree on branch bounty/<RUN_ID>/<BUNDLE>.
- Do NOT `git switch`, `git checkout`, or `git branch -D` to any other branch.
- Do NOT push, do NOT open PRs. The orchestrator handles all network operations.
- Do NOT write files outside this worktree. The only path outside the worktree you may write is $STATE_DIR, and only the specific filenames named in your prompt.
- If you finish with uncommitted changes, commit them on this branch before returning.
```

### Pre-dispatch guard

Before dispatching a wave of workers, verify:

1. Each worktree's branch is `bounty/<run_id>/<bundle>`.
2. Each worktree's base is `origin/main` (`git merge-base --is-ancestor origin/main <branch>`).
3. `MAIN_START_HEAD` still matches the main repo's current HEAD, and `MAIN_START_BRANCH` still matches the main repo's current branch.

Any mismatch aborts the wave and triggers the leak sweep.

### Post-phase leak sweep

After each phase that dispatches workers (planners, implementers, reviewers, PR-builders), re-run the 0c checks. If main-repo HEAD moved: STOP, surface to the user, do not auto-revert. If new files appeared under `CUR_DIRTY` that match bundle target files: move them to a `bounty-leak-<run_id>` branch (same pattern as moo-cleanup's WIP branch handling). Log a loud warning and continue — the user sees the leak branch in `/bounty-cleanup`.

Emit one leak-sweep one-liner per phase: `🧹 leak sweep @ <phase>: main unchanged, 0 leaks`.

## Live progress output

The orchestrator must surface progress as events happen, not just at phase boundaries. Silent waits feel broken.

Because Codex subagents do not stream writes into the shared `$STATE_DIR` while they are still running, the live boundary is different from Claude Code:

- Treat each completed subagent result as an event boundary.
- Serialize accepted claims, votes, and fix reports into `$STATE_DIR` immediately when that subagent finishes.
- Print the event line immediately after validation and persistence. Do not wait for the rest of the phase to finish.

Rules that apply to every phase below:

1. Wait on background subagents incrementally, not as one big blocking batch. Use `wait_agent` on the outstanding agent ids with a short timeout, process whichever agent finishes first, then continue with the remaining set.
2. Print a one-liner per event as it lands. Use compact formats:
   - New claim: `🛡️ Security/1 → BUG-003  high  sql-injection  app/Http/Orders.php:142`
   - Collision: `🛡️ Security/2 collided with 🛡️ Security/1 on BUG-003  (collision_count = 2)`
   - New vote: `BUG-003  ✓ VALID  from 🧵 Concurrency  (sanitizer bypassed upstream)`
   - Claim resolved: `BUG-003 CONFIRMED (4 VALID / 1 FP / 2 abstain)  finder +1 → 🛡️ Security`
   - Bundle planned: `📦 fix-acl  3 bugs  test_strategy=tdd  reviewer=gpt-5.4-mini`
   - Plan written: `📋 plans/fix-acl.md written  3 bugs  commit_order=[…]`
   - Fix committed: `BUG-003 (fix-acl) committed  (2 files, test added, phpunit ✓)`
   - Bundle reviewed: `fix-acl  3/3 APPROVE from 🧵  → kept, implementer +3`
3. Re-render the running leaderboard on every scoring change (new confirmation, false positive, or fix approval) **AND at the start of every phase** — even when nothing has scored yet. Use the exact table format from Phase 0e; prefix `🏆 BOUNTY CHAMPIONSHIP — standings @ phase N (run <RUN_ID>)`. Never substitute a text summary; the table is the contract. Also never substitute a `leaderboard.json` file-write preview for the table — the JSON is machine state, the ASCII table is the scoreboard. Emit the table as assistant stdout text right after every `leaderboard.json` update. Render Clawd (0f) above the table; pick the frame matching the most recent event (confirmation, FP, fix-approved, lead-change, or the quiet-tick frame when re-render was triggered by a phase boundary).
4. Never compress multiple new events into a single summary line. If three claims are accepted from one hunter result, print three claim lines.
5. If no agent completes for roughly 20-30 seconds, emit a short heartbeat update that names the phase and the remaining agent count, then keep waiting.
6. Refresh `$STATE_DIR/README.md` whenever the visible leaderboard changes so the on-disk dashboard matches what the user sees.

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
- the **dismissed list** (contents of `$STATE_DIR/dismissed.yml` — see Phase 0d) inlined verbatim. Instruction: "Before filing a claim, scan this list; if the claim matches an entry's file-glob *and* category, do not file it unless you have strong evidence the rationale does not apply (e.g. the code has changed). If you override a dismissal, include `override_reason` in the claim JSON." Skip this when the list is empty or `--ignore-dismissed` was set.
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
- writes accepted claims to `$STATE_DIR/claims/BUG-<lens>-<seed>-<seq>.json`
- merges duplicates by fingerprint and records later submissions in `co_discovered_by`
- increments `collision_count` on the surviving claim each time a duplicate lands; emits a `collided` one-liner live
- runs a **secondary semantic-dedup pass** over surviving claims: buckets by `(normalize(file), line // 10)` ignoring category, and for any bucket with ≥2 claims from different specialties, merges them (keep highest-severity as survivor, append others' finders to `co_discovered_by`, increment `collision_count`, record merged categories). Emits a semantic-merge one-liner, e.g. `🔗 semantic merge: BUG-authz-1-001 + BUG-security-1-004 → surviving BUG-security-1-004`
- enforces `--max-claims` by keeping the highest-severity claims first

## Phase 3: Voting

### Pre-filter against the dismissed list

Before dispatching voters, walk every surviving claim and test it against the dismissed entries loaded at Phase 0d. A claim matches when both its file matches an entry's glob *and* its category overlaps an entry's category token.

For each match:

1. Write a synthetic `FALSE_POSITIVE` tally into `$STATE_DIR/tally.json` with `resolution_source: "dismissed-list"` and `dismissed_entry_id: "<id>"`.
2. Do **not** dispatch voters on that claim.
3. Do **not** apply the −3 finder penalty — the hunter was given the list and a policy call should not tank a specialist's score. Record `points_awarded: 0` with reason `dismissed-policy`.
4. Log one-liner: `📒 auto-FP: <BUG-ID>  matches <entry-id> (<entry-title>)`
5. Re-render the leaderboard.

Skip this step when the dismissed list is empty or `--ignore-dismissed` was set.

### Persist new false positives at end of Phase 3

After voting settles, batch-prompt the user for every `FALSE_POSITIVE` whose `resolution_source` is `voter` (not `dismissed-list`). Reply options: `all`, `pick <ids>`, `none`, `edit`. Default `all` on 30s timeout. On confirm, append entries to the live `.bounty/dismissed.yml` (atomic temp-file + rename) with a best-effort `title` and `rationale` composed from voter reasons.

Skip when `persist_dismissed: false` was resolved or zero voter-FPs surfaced this run.

### Voting modes

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
- model `$MODEL_VOTE` (default `gpt-5.4-mini`)

### Live progress during voting

Do not wait for a whole claim batch to finish before showing progress.

Instead:

1. Track outstanding voter agent ids for the active batch.
2. Use `wait_agent` with a short timeout and process each completed voter as soon as it returns.
3. Append that vote immediately to `$STATE_DIR/votes/<BUG-ID>.jsonl` and print the new-vote one-liner.
4. As soon as all votes for a given claim are present, resolve that claim immediately, update the leaderboard, print the claim-resolved one-liner, and re-render the leaderboard table.
5. Continue until the batch has no unresolved claims left, then move to the next batch.

The orchestrator appends every vote to `$STATE_DIR/votes/<BUG-ID>.jsonl`, tallies the result, and updates `$STATE_DIR/leaderboard.json` using the thresholds from `../../skills/bounty/references/scoring-rules.md`.

Do not wait for the end of the batch to update the visible standings. The leaderboard update happens claim-by-claim.

## Phase 3.5: Proof-of-concept (high/critical confirmed only)

Skip when `--poc off` or `--no-fix` is set.

For every CONFIRMED claim whose severity qualifies (`auto` = ≥ high; `always` = any), dispatch one PoC subagent per claim. PoCs reproduce the bug before the fix phase spends its biggest model on it; a PoC that fails to reproduce demotes the claim to FALSE_POSITIVE.

### Worktree contract

PoC agents run exploit code, so their contract is stricter than planners/implementers:

```
You are running a PoC for BUG-<id> in an isolated read-only worktree.
- Do NOT write any file under the worktree (the repo checkout). Treat it as read-only.
- Do NOT run `git add`, `git commit`, `git switch`, `git checkout`, or any branch operation.
- Do NOT push anything anywhere.
- The ONLY paths you may write to are under $POC_DIR ($STATE_DIR/pocs/<BUG-ID>/) and
  the single file $STATE_DIR/pocs/<BUG-ID>.json.
- If reproducing requires running the application, invoke it from $POC_DIR with
  cwd=$POC_DIR — never from the worktree root. Copy (don't symlink) any sample
  inputs you need into $POC_DIR first.
```

After the PoC returns, the orchestrator runs `git -C <worktree> status --porcelain`. Dirty worktree → PoC rejected with `status: "contract_violation"`, claim unchanged (no bonus, no demotion), loud warning in the final report. The worktree is discarded and a fresh one cut from `origin/main` before the next phase.

### Dispatch

For each eligible CONFIRMED claim, spawn one read-only worktree subagent (`agent_type: "worker"`, model `$MODEL_HUNT` — sonnet-tier). Run all eligible PoCs in parallel, batched in groups of 8, with a 10-minute wall-time cap per PoC. Each agent's prompt includes the claim JSON, the cited file ± 20 lines of context, the recon report, the repo's run/validation commands, `$POC_DIR`, `$STATE_DIR`, and the contract block above verbatim.

### Output contract

Each PoC writes:

1. `$STATE_DIR/pocs/<BUG-ID>/` — artifacts (script, payload, captured output). Never committed.
2. `$STATE_DIR/pocs/<BUG-ID>.json` — verdict (see `../../skills/bounty/references/state-schema.md#pocsbug-idjson` for the schema).

Required fields on the JSON: `bug_id`, `reproduced`, `method`, `observed`, `expected_after_fix`. Missing any → `status: "malformed"` and the claim is treated as if `--poc off` for this bug.

### Resolution

| PoC outcome | Effect |
|-------------|--------|
| `reproduced: true` | Claim stays CONFIRMED, finder gets **+1 PoC bonus**, `claim.poc: true` flows into Phase 4 and the PR body |
| `reproduced: false` | Claim demoted to FALSE_POSITIVE, finder gets standard **−3**. PoC rationale feeds the voter-confirmed-FP prompt at end of Phase 3 |
| `contract_violation` / `malformed` | Claim unchanged, flagged in final report |

Re-render the championship table after each PoC resolves. Print a one-liner per PoC:

```
🧪 PoC/BUG-security-1-001  REPRODUCED  method=sqli-via-xff  finder +1 → 🛡️ Security
🧪 PoC/BUG-authz-1-007     DID-NOT-REPRODUCE  DEMOTED to FP  finder −3 → 🛡️ AuthZ
🧪 PoC/BUG-resources-1-002 CONTRACT VIOLATION (wrote to worktree) — quarantined
```

PoC-demoted claims are folded into the same end-of-Phase-3 dismissed-list prompt as voter-confirmed false positives, tagged `[poc-demoted]` so the user sees which dismissals came from code-reading voters vs. from an exploit that failed to run.

## Phase 4: Bundled fixes

Skip this phase when `--no-fix` is set.

Instead of one fixer per bug, Phase 4 runs a **plan → implement → review** pipeline over bundles. For 30+ confirmed bugs this is the difference between ~90 model calls and ~18.

### Volume gate

Before dispatching planners, check `confirmed_count` against `--max-fixes` (default 10). If exceeded, ping the user with the bundle plan and **wait up to 3 minutes** — do not block indefinitely. Options: `all` / `high-only` / `pick <slugs>` / `abort`. On reply, honor the choice and record `user_fix_scope` in `config.json`. On timeout, record `user_fix_scope: "all"` with `gate_timeout: true`, announce the timeout, and continue with every bundle. Skip the gate entirely when `confirmed_count ≤ max_fixes`.

Severity policy:

- `low`, `medium`, `high`: eligible for bundled fixes
- `critical`: do not auto-fix; open a GitHub issue if GitHub access is available, otherwise record the blocker clearly in the final report

### Step 4a: Bundle confirmed bugs

Cluster fixable bugs and write `$STATE_DIR/bundles.json`. Default heuristic:

1. One bundle per lens with ≥2 confirmed bugs.
2. Lone-bug lenses merge into a `misc` bundle alongside adjacent lenses.
3. Cap at `--max-bundles` (default 6); merge smallest first when exceeded.
4. Each bundle has a kebab-slug name (e.g. `fix-acl`, `fix-fail-open`, `fix-concurrency`, `fix-nullsafety-misc`).

**Integrity check before proceeding.** After writing `bundles.json`, verify every `bug_id` in every bundle exists as a file under `$STATE_DIR/claims/`. If any referenced id is missing, the state is polluted (most often from a mid-run restart with stale `bundles.json` from a prior run). Abort Phase 4 with a clear message naming the missing ids and recommending `/bounty --fresh` or `/bounty-cleanup`.

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

The orchestrator writes the plan to `$STATE_DIR/plans/<bundle>.md` and prints a plan-written one-liner.

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
- return a per-bug report the orchestrator serializes to `$STATE_DIR/fixes/<BUG-ID>.json` (the only state path any worker is allowed to imply)

The orchestrator owns integration:

- apply the returned patch/edits in the bundle's worktree on branch `bounty/<run_id>/<bundle>`
- run project validation after each applied bug
- write `$STATE_DIR/fixes/<BUG-ID>.json` with the committed status and print the fix-committed one-liner
- commit on the bundle branch only when the per-bug validation passes

## Phase 5: Bundle review and ship

For each completed bundle, dispatch **one reviewer subagent** — not the planner or implementer of that bundle, and not a bug's original finder when avoidable:

- `gpt-5.4-mini` when the bundle's plan frontmatter says `test_strategy: tdd`
- `gpt-5.4` when `test_strategy: architectural`

**The reviewer is blind.** It receives only:

1. The bug claim JSON(s) in the bundle
2. Per-commit diffs (`git log -p origin/main..bounty/<run_id>/<bundle>`) and the commit messages
3. Project rules (`CLAUDE.md` / `AGENTS.md`)
4. For any bug with a reproduced PoC, the `method` / `observed` / `expected_after_fix` fields from `$STATE_DIR/pocs/<BUG-ID>.json`

And explicitly **not** the plan (`$STATE_DIR/plans/<bundle>.md`), the implementer's `$STATE_DIR/fixes/<BUG-ID>.json` summaries, or any natural-language explanation of what the bundle was supposed to do beyond the per-commit messages themselves. Rationale: a reviewer who reads the planner's "why" first will rationalize a weak diff; a blind reviewer judges whether the commit + test speaks for itself, which is the standard bounty wants to enforce.

The reviewer returns one verdict per bug (`APPROVE` / `REJECT` with a one-sentence reason). The orchestrator writes `$STATE_DIR/reviews/<bundle>.json`.

Process reviewers incrementally via `wait_agent`; the moment a bundle's review lands, resolve it and re-render the leaderboard.

Resolution per bug:

- `APPROVE`: keep the commit; implementer gets `+1`
- `REJECT`: revert that single commit on the bundle branch; the bug goes back to the queue for one isolated per-bug retry (`gpt-5.4`), then abandoned if it rejects again

After review, ship approved commits across **multiple PRs** — never one mega-PR. A single PR with 30+ bugs and 1,500+ changed lines is unreviewable.

### Ship limits

| Threshold | Default | Override |
|-----------|---------|----------|
| Max bugs per PR | 10 | `--max-bugs-per-pr` |
| Max added+removed lines per PR | 400 | `--max-lines-per-pr` |

Either threshold triggers a split. Exception: a single commit that alone exceeds `--max-lines-per-pr` gets its own PR marked `<module>: large fix — <theme>` so reviewers know what they're looking at.

### Step 5a: Group approved commits into PR batches

Build `$STATE_DIR/pr-groups.json`:

1. Start with the approved per-bug commits across bundle branches, preserving each bundle's `commit_order`.
2. Group greedily by bundle theme first, then by severity.
3. Measure `bug_count` and `est_lines = Σ (additions + deletions)` per commit (`git log --numstat <bundle-branch>..origin/main`).
4. When adding the next commit would exceed either threshold, close the group and open the next. Keep a bundle's commits together when they fit.
5. Name each group `bounty/<run_id>/pr-<N>-<slug>`.

Print a one-liner per group so the user sees the plan before any branch is pushed.

### Step 5b: Build each aggregate branch

For each group: create `<slug>` off `origin/main`, cherry-pick the approved per-bug commits from their bundle branches preserving order, run aggregate validation. If validation fails, bisect, drop the offending commit, and record the drop in `pr-groups.json`. Never squash — the individual commits are the audit trail.

Run a leak sweep (see Branch isolation contract) after all aggregates are built.

### Step 5b.5: Pre-push secrets scan

Skip when `--no-secrets-scan` is set. Default: on.

Before pushing any aggregate branch, scan its `origin/main..HEAD` diff against the pattern list in `../../skills/bounty/references/secret-patterns.txt` (AWS keys, GitHub PATs, Slack tokens, private-key blocks, JWTs, and generic high-entropy assignments). The orchestrator reads the file at Phase 0 and snapshots its SHA-256 into `config.json` as `secret_patterns_sha256` so a mid-run edit to the pattern file doesn't change scan behavior for this run.

A hit blocks **only that PR group** — other groups still ship if they're clean. Record the hit on `pr-groups.json` under the group's `blocked_by_secrets_scan` field:

```json
{
  "slug": "bounty/<run_id>/pr-2-null-safety",
  "status": "blocked",
  "blocked_by_secrets_scan": [
    {"commit": "a1b2c3d", "file": "tests/fixtures/config.php", "line": 17, "pattern": "aws-secret-key"}
  ]
}
```

**Never write the matched secret value anywhere** — only the pattern name, file, and line. `pr-groups.json` is sometimes attached to bug reports or shared in screenshots.

Print a one-liner per blocked group:

```
🚫 PR 2/3  pr-2-null-safety  BLOCKED by secrets-scan  tests/fixtures/config.php:17 (aws-secret-key)
```

On a hit the underlying bug's finder is not penalized — the fix that leaked the secret was the implementer's work. The implementer's `+1` fix credit is reversed and the bug returns to the queue for one isolated per-bug retry. If the retry also hits the scan, abandon the bug and surface it in the final report.

### Step 5c: Push and open PRs

Use `gh` or the GitHub connector to push each aggregate branch and open its PR. Body uses the template from `../../skills/bounty/references/scoring-rules.md` and cross-links the siblings.

**PR titles must read like normal product work** — lead with the module, then a positive benefit statement (imperative-past). Do NOT mention "bounty", "bounty sweep", "bounty run", or numeric bug counts in the title. The bounty framing belongs in the body, not the title. See `../../skills/bounty/references/scoring-rules.md#title-style--positive-module-generic-no-bounty-branding` for worked examples. Summary: `smartcart: Reduced risk and improved reliability` — not `smartcart: Resolved 11 issues from bounty sweep`. The same rule applies if you pass `--subject` when merging (`gh pr merge … --squash --subject …`) — mirror the PR title style; never reintroduce bounty branding.

```
> Part N of M from bounty run <run_id>.
> Siblings: #<pr-1>, #<pr-2>, #<pr-3>.
```

Record each PR url in `$STATE_DIR/pr-groups.json` and `$STATE_DIR/README.md`.

### Step 5d: Critical issues

Open a separate GitHub issue per `critical` bug instead of committing. Link each issue from the final report.

If GitHub tooling is unavailable, stop after local commits and report the exact blocker — do not fall back to a single combined branch.

### Step 5e: Automated PR review feedback loop

Modeled on `/moo`'s `moo-pr-feedback-handler` from the Moogento repo. After every PR opens in Step 5c, CI + bot reviewers (GitHub Advanced Security, CodeRabbit, Copilot, Claude review bot, etc.) typically post within 5–15 min. Step 5e reads those comments, validates them, fixes the valid ones in-place on the PR branch, and — when `pr_review_replies` is `true` — replies with a rationale to any that are wrong. Loop up to `--pr-review-rounds` rounds per PR (default 2).

Skip Step 5e entirely when `--no-fix` was passed (no PRs exist) or when Step 5c opened zero PRs.

Dispatch one worker per PR in parallel — each PR has its own worktree on its `bounty/<run_id>/pr-*` branch and its own `$STATE_DIR/pr-review/<pr-number>.json` state file. Each worker's prompt includes `PR_NUMBER`, `PR_BRANCH`, worktree path, `STATE_DIR`, `MAX_ROUNDS = --pr-review-rounds`, `WAIT_SECONDS = --pr-review-wait`, `POST_REPLIES = config.pr_review_replies`, and the repo's validation commands (from `CLAUDE.md` / `AGENTS.md`, cached in `config.json`).

Per-round loop (1…MAX_ROUNDS):

1. **Wait for checks.** Poll `gh pr checks $PR_NUMBER` every 60s up to `WAIT_SECONDS`, breaking early when no check is `pending / in_progress / queued / running`. Record failed checks for the final report.

2. **Read new comments.** Both issue-comments and review-comments; skip ids already in `processed_comment_ids` from prior rounds. If zero new actionable comments, exit the loop with `exit_reason: "no_new_feedback"`.

3. **Evaluate each suggestion** by reading the referenced file and ±10 lines of context. Label `VALID`, `INVALID`, or `SKIP`. Never blind-apply. A technically-correct suggestion that conflicts with a documented ADR / comment / test is `INVALID` with a pointer.

4. **Implement valid fixes in-place** on the PR branch. Keep fixes minimal. Run validation after every fix; if validation regresses, roll the fix back and record `{"verdict": "VALID", "applied": false, "reason": "validation regressed"}` — the suggestion still goes into the rejection reply (if replies are on).

5. **Post rationale reply for INVALID** (one consolidated comment per round) **only when** `POST_REPLIES == true` and at least one `INVALID` was recorded this round. Body:

   ```
   ## Bounty automated-review response — round <N>

   The following suggestions were evaluated by the bounty run and not applied:

   - **<file>:<line>** — <what was suggested> — <why rejected>

   These points have been reviewed and will not be addressed in this PR.
   ```

   Skip the reply entirely when `POST_REPLIES` is false or when every new comment was `VALID` / `SKIP`. Either way, the worker still records every INVALID verdict + rationale in `$STATE_DIR/pr-review/<pr-number>.json` so the final report can surface them.

6. **Commit and push** fixes as a single combined commit per round. Follow the repo's commit rules (no `fix:` / `feat:` prefix, lowercase module, no AI attribution):

   ```
   <module>: improved code quality from automated review (round <N>)

   Applied valid suggestions from the round-<N> automated PR review.

   Changes:
   - <specific fix 1>
   - <specific fix 2>

   PR #<pr-number>, bounty <run_id>
   ```

7. **Repeat** until `MAX_ROUNDS`. Stop early when a round produces zero new actionable comments or every new comment was `INVALID` / `SKIP`.

Live stdout per round:

```
🤖 PR #1234 round 1: 7 new comments · VALID 4 · INVALID 2 · SKIP 1
✍  PR #1234 round 1: 4 fixes pushed (commit a1b2c3d), rejection reply posted for 2
```

Branch isolation still applies: the worker only touches its own PR branch, never pushes elsewhere, never writes files outside its worktree or `$STATE_DIR`. Leak-sweep after every worker returns.

## Phase 6: Final report

Render final standings to stdout and `$STATE_DIR/README.md`, including:

- confirmed bugs
- false positives
- fixes approved
- abandoned fixes and reasons
- PR URL if created
- critical issues opened
- **automated-review outcomes** per PR (from `$STATE_DIR/pr-review/<pr-number>.json`): rounds run, fixes applied with commit SHAs, rejections with rationale, whether `pr_review_replies` was on or off
- rough wall-clock and cost notes

## Notes

- Discovery should stay parallel; integration can become serial if that is the safer way to preserve correctness.
- Do not silently skip phases because Codex and Claude have different subagent mechanics. Adapt the orchestration, not the workflow.
- The Codex port should feel live even though the mechanics differ: process the first finished subagent immediately, not the whole wave at once.
