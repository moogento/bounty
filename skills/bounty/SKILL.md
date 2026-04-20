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
| `--poc <mode>` | `auto` | Phase 3.5 proof-of-concept gate. `auto` = run for every CONFIRMED claim with severity ≥ `high`. `always` = run for every CONFIRMED claim regardless of severity. `off` = skip the phase entirely. A PoC that fails to reproduce demotes the finding to FALSE_POSITIVE (finder −3) |
| `--secrets-scan` / `--no-secrets-scan` | on | Step 5b.5 scans each aggregate branch's diff for credential patterns before `gh pr create`. A hit blocks that PR group, records the finding in `pr-groups.json`, and surfaces it in the final report. Off only for debugging false positives |
| `--no-fix` | false | Skip Phase 4–6, just report findings |
| `--voting-mode <mode>` | `auto` | `full` = N−1 voters per claim (small runs). `panel` = 1 voter per specialty reviews every non-own claim (8 agents total, independent of claim count). `auto` = `full` when claims ≤ 20, `panel` otherwise |
| `--max-fixes N` | 10 | If confirmed count exceeds this, Phase 4 pauses and asks the user to confirm/curate/reject the bundle plan before dispatching planners |
| `--model-hunt <tier>` | `sonnet` | Model tier for hunters |
| `--model-vote <tier>` | `haiku` | Model tier for voters; orchestrator auto-falls back to `sonnet` on haiku rate-limit errors (announced) |
| `--model-plan <tier>` | `opus` | Model tier for bundle planners |
| `--model-fix <tier>` | `sonnet` | Model tier for bundle implementers |
| `--max-bundles N` | 6 | Cap on Phase-4 bundles. Smallest lenses merge into a `misc` bundle when exceeded |
| `--max-bugs-per-pr N` | 10 | Phase-5 splits into an additional PR when a group would exceed this bug count |
| `--max-lines-per-pr N` | 400 | Phase-5 splits into an additional PR when a group's added+removed line count would exceed this |
| `--pr-review-rounds N` | 2 | Step 5e polls each opened PR for automated-review comments, validates them, fixes the valid ones, and (when replies are on) replies with a rationale to the invalid ones, up to N rounds per PR |
| `--pr-review-wait <secs>` | 300 | How long to wait after opening a PR before the first round of feedback polling, giving CI + bot reviewers time to post |
| `--pr-review-replies` / `--no-pr-review-replies` | ask | Whether Step 5e posts a consolidated rationale comment on each PR for suggestions marked `INVALID`. Explicit flag skips the Phase-0 verify; when neither is set, the orchestrator verifies with the user at Phase 0 (default `on` after 30s timeout). Records the resolved choice in `config.json` as `pr_review_replies` |
| `--dismissed-file <path>` | `.bounty/dismissed.yml` | Per-repo memory file of patterns previously confirmed NOT to be bugs. Loaded at Phase 0 and passed to every hunter + used to auto-dismiss matching claims before voting |
| `--persist-dismissed` / `--no-persist-dismissed` | ask | Whether to offer appending voter-confirmed false positives to the dismissed file at end of Phase 3. Explicit flag skips the prompt; neither flag → ask the user when at least one FP lands (default `on` after 30s timeout) |
| `--ignore-dismissed` | false | Debugging/rediscovery: skip the dismissed-file load entirely this run. Hunters re-surface patterns that would normally be filtered |
| `--fresh` | false | At Phase 0, proceed alongside any existing `.temp/bounty/<id>/` dirs without prompting (each run uses its own state dir) |
| `--resume <id>` | — | At Phase 0, reuse an existing `.temp/bounty/<id>/` state dir and continue from its last completed phase. `--resume` with no id resumes the sole in-progress run when there is exactly one |

If no scope is given, ask the user once whether to scope the hunt.

## Phase 0: Initialize state

Phase 0 is five gates. Do them in order. Do not skip any to "save time": every gate below was added because a past run skipped it and produced state pollution, branch collisions, or commits on the wrong branch.

### 0a. Generate the run id

Every run gets a unique id. Branch names, worktree labels, and (optionally) the state subdirectory are scoped under it so two concurrent bounty runs in the same repo never collide.

```bash
SCOPE_SLUG=$(echo "${SCOPE:-repo}" | tr '/' '-' | tr -cd '[:alnum:]-' | sed 's/^-*//;s/-*$//' | cut -c1-24)
RUN_ID="$(date -u +%Y%m%d-%H%M%S)-${SCOPE_SLUG}"
```

Record `run_id` in `config.json`. Every branch created by this run MUST be named `bounty/<run_id>/<bundle>` (Phase 4 and Phase 5 use this; see the Branch isolation contract below).

### 0b. Scan for existing runs

Each run's state lives under its own per-run directory: `.temp/bounty/<run_id>/`. This makes concurrent bounty runs in the same repo safe — they never share a state dir, and `/bounty-cleanup` can target one run without touching the others.

Scan for any in-progress runs (at least one `.temp/bounty/<id>/config.json` whose `last_completed_phase < 6`):

```bash
EXISTING_RUNS=$(find .temp/bounty -mindepth 2 -maxdepth 2 -name config.json 2>/dev/null)
```

Branch on what's there:

| Condition | Action |
|-----------|--------|
| `--resume <id>` given, that run exists | Load its `config.json`, adopt its `run_id`, skip to the first incomplete phase |
| `--resume` with no id, exactly one in-progress run exists | Resume that one |
| `--resume` with no id, multiple or zero in-progress | List the in-progress runs and abort with a message |
| `--fresh` given, matching-scope runs exist | Leave them alone; just create this run's own `.temp/bounty/<run_id>/` |
| No flag, at least one in-progress run overlaps this run's scope | Surface them (`<id> · last-completed=<phase> · scope=<path>`). Ask: `resume-<id> / proceed-alongside / abort`. Default `proceed-alongside` on 30-second timeout — concurrent-by-default is the supported workflow |
| Nothing in-progress | Proceed |

"Overlap" means the existing run's `scope` is a prefix of, or prefixed by, this run's scope. Non-overlapping scopes (e.g. one run on `app/Foo`, another on `app/Bar`) are always safe to run alongside without prompting.

Do **not** write into another run's state directory. The per-run layout makes this physically impossible as long as every orchestrator write goes through `$STATE_DIR`.

### 0c. Working-tree and branch gate

```bash
MAIN_REPO="$(git rev-parse --show-toplevel)"
MAIN_START_BRANCH="$(git -C "$MAIN_REPO" branch --show-current)"
MAIN_START_HEAD="$(git -C "$MAIN_REPO" rev-parse HEAD)"
MAIN_START_DIRTY="$(git -C "$MAIN_REPO" status --porcelain | grep -vE '^\?\? \.temp/bounty' || true)"
```

Record all three in `config.json` under `main_start`. They are the reference points Phase 4 and Phase 5 use to detect leaked writes (see Branch isolation contract).

Block and prompt when any of the following is true:

1. `MAIN_START_BRANCH` is not `main` (or the repo's configured default). A bounty run's commits must land on fresh `bounty/<run_id>/*` branches cut from `origin/main`, not on top of whatever the user is currently working on.
2. `MAIN_START_DIRTY` is non-empty and at least one dirty path falls under `--scope`. This is exactly the overlap that's caused past runs to stop mid-phase asking about 22 uncommitted files.

Present the dirty files and ask: `stash-and-proceed / commit-first / proceed-anyway / abort`. On `stash-and-proceed`: `git stash push --include-untracked -m "bounty-<run_id>: pre-run stash"` and record the stash ref in `config.json` so `/bounty-cleanup` can surface it later.

### 0d. Initialize state directory

State is per-run, under `.temp/bounty/<run_id>/`:

```bash
STATE_DIR="$MAIN_REPO/.temp/bounty/$RUN_ID"
mkdir -p "$STATE_DIR"/{claims,votes,fixes,plans,reviews,pr-review}
```

Write `$STATE_DIR/config.json` with the resolved arguments *and* `run_id`, `state_dir: "<absolute path>"`, and `main_start`. Every downstream agent prompt must include `STATE_DIR=<absolute>` and `RUN_ID=<run_id>` and instructions to write every output file under `$STATE_DIR/...` — never a relative `.temp/bounty/...` path, which could land in a different run's directory if another bounty is active.

Update the convenience pointer so humans can always find the latest run:

```bash
ln -snf "$RUN_ID" "$MAIN_REPO/.temp/bounty/latest"
```

**Seed `$STATE_DIR/leaderboard.json` with every specialist in the `specialists` list, not just the ones we expect to find bugs.** A recent run initialized only 5 specialists and the leaderboard silently dropped the other 3 when they submitted claims. Iterate the resolved specialist list; write a zeroed row for each one.

**Write `leaderboard.json` (and every subsequent update) via `Bash` + single-line minified JSON — not the `Write` tool.** The `Write` tool renders a multi-line JSON preview in the Claude Code UI (observed: a 13-line `Write(leaderboard.json)` block dominating the visual frame and burying the championship table that was supposed to be the scoreboard). The JSON file is machine state; the ASCII table (0e) is the user-facing artifact. Keep the file write quiet so the table can be seen. Pattern — atomic minified write:

```bash
tmp="$(mktemp "$STATE_DIR/leaderboard.json.XXXX")"
jq -cn --arg at "$(date -u +%FT%TZ)" --argjson scores '{"security":{"found":0,"confirmed":0,"fp":0,"collisions":0,"fixes":0,"score":0}, …}' \
  '{updated_at:$at, scores:$scores}' > "$tmp" && mv "$tmp" "$STATE_DIR/leaderboard.json"
```

The same rule applies at every later update (Phase 3 resolutions, Phase 4 approvals): single-line JSON via Bash, never the Write tool.

**Resolve `pr_review_replies` before Phase 1.** Step 5e optionally posts a consolidated rationale comment on each PR explaining why any `INVALID` suggestion was rejected. Because this writes to a user-visible place (the PR timeline), the default is to verify — but only when the user did **not** already express a preference on the command line:

| Command-line input | Resolved `pr_review_replies` | Prompt? |
|--------------------|------------------------------|---------|
| `--pr-review-replies` | `true` | No |
| `--no-pr-review-replies` | `false` | No |
| `--no-fix` (no PRs will exist) | `false` | No |
| neither flag, `--no-fix` absent | ask the user | Yes |

When prompting, print:

```
🤖 Step 5e can reply on each PR with a rationale for any suggestion it marks INVALID.
   Post these rationale replies?  (y/n)  — default y in 30s
```

Wait up to 30s. On `y` / timeout: `pr_review_replies: true`. On `n`: `pr_review_replies: false`. Record the resolved value in `config.json` so a later `--resume` does not re-prompt. Announce the resolved value once: `✓ PR rationale replies: on` or `… off`.

Step 5e still evaluates every comment and still records INVALID verdicts in `$STATE_DIR/pr-review/<pr-number>.json` regardless — the flag only controls whether the rationale comment is posted to GitHub. The final report always summarizes what would have been replied to, so the user sees the reasoning even when posting is off.

**Load the per-repo dismissed-bug memory.** A repo can keep a list of patterns already determined NOT to be bugs. The bounty skill persists and reads this list so the same false-positive doesn't keep showing up run after run. Path defaults to `--dismissed-file` (`.bounty/dismissed.yml` at the repo root — outside `$STATE_DIR`, so it is shared across runs and lives under source control).

Schema (`version: 1`):

```yaml
version: 1
dismissed:
  - id: MOO-001
    added: 2026-04-20
    run_id: 20260420-030000-smartcategories   # optional — the run that first surfaced it
    files: ["app/code/Moogento/**/view/adminhtml/templates/system/config/hint.phtml"]
    categories: ["security", "xss", "supply-chain"]
    title: Raw HTML in Moogento hint templates
    rationale: |
      Moogento controls the upstream content; raw HTML is intentional for
      formatting. The Moo.php null guard is the relevant safeguard.
```

Load the file at Phase 0 (skip if missing or `--ignore-dismissed`). Copy its contents into `$STATE_DIR/dismissed.yml` so the run's state dir is self-contained; the orchestrator reads from `$STATE_DIR/dismissed.yml` thereafter. Announce once at Phase 0:

```
📒 dismissed list: 4 entries loaded from .bounty/dismissed.yml
```

A claim **matches a dismissed entry** when both are true:

- the claim's file matches any glob in `entry.files` (fnmatch-style, e.g. `app/code/Moogento/**/hint.phtml`)
- the claim's category (case-insensitive substring, or a word in the claim's category) overlaps any token in `entry.categories`

Matching is used in two places:

1. **Phase 2 (hunt)** — every hunter prompt includes the full `dismissed` list inline. Instruction: "Before filing a claim, scan this list; if the claim matches an entry, do not file it unless you have strong evidence overriding the rationale. If you override, include `override_reason` in the claim JSON; orchestrator will still pre-check."
2. **Phase 3 start** — after hunting, before voting dispatch, the orchestrator iterates surviving claims and auto-marks any matching claim as `FALSE_POSITIVE` with `resolution_source: "dismissed-list"` and a pointer to the entry id. **No finder penalty** for these — the hunter saw the list and the hunter's judgment should not be tanked by a policy call. Log one-liner per auto-dismissal:

   ```
   📒 auto-FP: BUG-security-1-003  matches MOO-001 (Raw HTML in Moogento hint templates)
   ```

   Write the auto-FP result to `$STATE_DIR/tally.json` with the standard shape so the leaderboard renders normally. These claims do NOT go to voters — they skip Phase 3 voting entirely.

3. **End of Phase 3** — any claim that actually went to vote and came back `FALSE_POSITIVE` is a candidate for the dismissed list. If `persist_dismissed` resolves to `true`, batch-prompt the user:

   ```
   📒 3 new false positives surfaced this run. Persist to .bounty/dismissed.yml so future runs skip them?
      [1] BUG-security-1-007  app/Controller/Orders.php:142  "SQL injection via X-Forwarded-For"  — voters: 5 FP / 0 VALID
      [2] …
      Reply:  (a) all  (b) pick 1,3  (c) none  (d) edit — default (a) in 30s
   ```

   On `all` / timeout: append one entry per FP to `.bounty/dismissed.yml` with a best-effort `title` and `rationale` derived from voter reasons. On `pick`: only the chosen ids. On `edit`: open $EDITOR with a prefilled template. On `none`: skip.

Resolve `persist_dismissed` the same three-state way as `pr_review_replies`:

| Command-line input | Resolved | Prompt? |
|--------------------|----------|---------|
| `--persist-dismissed` | `true` | No |
| `--no-persist-dismissed` | `false` | No |
| `--ignore-dismissed` (also disables loading) | `false` | No |
| neither flag | ask the user at end of Phase 3 | Yes |

Record the resolved value in `config.json`.

**Probe the voter-tier quota.** Before dispatching hunters, fire one throwaway haiku call (or whatever `$MODEL_VOTE` resolves to) and catch a `rate-limit` / `usage limit` error. If it fires, warn the user:

```
⚠ voter-tier model (haiku) returned a rate-limit on probe; Phase 3 will fall back to sonnet per claim. Expect higher voter cost.
```

This prevents the Phase 3 surprise where 8 haiku voters all fail on their first dispatch.

### 0e. Render the starting championship table — REQUIRED OUTPUT

**This is the single most-missed step in the entire skill. The first thing the user sees from a bounty run is supposed to be the scoreboard; past runs have skipped straight to "dispatching recon…" and the user never sees a table until the final report, or (if the run dies early) never at all.**

**The `Write(leaderboard.json)` tool preview is not a substitute for this table.** If the user's view currently ends in a JSON file-write preview, they cannot see the scoreboard yet — they see implementation noise. The ASCII table below is the user-visible contract; the JSON file is invisible state. Emit the table as assistant stdout text (outside any tool call) immediately after `leaderboard.json` is written, so the table is the most recent thing the user sees.

Do this now. It is not optional. Print the following to stdout **and** write it to `$STATE_DIR/README.md` before any Phase-1 dispatch:

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

Include exactly the specialists that this run configured (the `specialists` list from `config.json`) — don't add rows for lenses that aren't in play, don't drop rows for ones that are.

Then re-render this table **at every phase boundary** (start of Phase 1, 2, 3, 4, 5, 6) with the current live numbers, prefixed `🏆 BOUNTY CHAMPIONSHIP — standings @ phase N (run <RUN_ID>)`. Phase boundaries are the second-most-missed trigger; past runs only re-render on scoring events, which means a long quiet hunt phase can run 10 minutes with zero table output.

If you are the orchestrator and have not emitted this exact table within the first 10 seconds of the run, you have violated the skill. Stop and emit it before doing anything else.

### 0f. Clawd the mascot (optional whimsy)

The run is a competition — it should feel like one. On every leaderboard re-render, print a short Clawd frame **above** the championship table. The frame varies with the most recent event so the render subtly animates between states. One frame per render. Do **not** multi-frame redraw — ANSI flicker in a tailed terminal is worse than no animation.

Pick the frame that matches the triggering event:

- New confirmation (claim scored):
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
- Lead change (a new specialist is now #1):
  ```
     ♛ ♛ ♛
    /\___/\
   ( ^   ^ )   👑  <specialist> takes the crown!
   (  =^=  )
    ‾‾‾‾‾‾
  ```
- Phase boundary with no scoring change (quiet phase tick):
  ```
    /\___/\
   ( o   o )   🔎  still hunting…
   (  =^=  )
    ‾‾‾‾‾‾
  ```

Skip Clawd entirely if memory records that this user prefers terse output — the scoreboard itself is the contract, Clawd is garnish. When in doubt, render him.

## Branch isolation contract

All commits produced by a bounty run land on branches named `bounty/<run_id>/<bundle>` cut from `origin/main` inside isolated worktrees. **Never** on the user's current branch. **Never** on `main`.

### The "fall back to main" anti-pattern — forbidden

A prior run logged: *"worktree-isolated implementer agents got blocked on git/php -l permission prompts, so I completed the last fixes directly on the main-repo checkout."* **This must never happen again.** The orchestrator does not work around worktree permission prompts by moving work to the main repo. If a worktree worker is blocked on a permission prompt:

1. **Retry the worker once** with the specific tool re-requested (e.g. `Bash(php:*)`, `Bash(git:*)`) pre-allowed in the dispatch. If the retry still blocks, proceed to step 2 — do not try a third time.
2. **Mark the bundle blocked, not complete.** Write `$STATE_DIR/fixes/<BUG-ID>.json` with `{"status": "blocked", "reason": "worktree tool prompt: <tool>"}` for every remaining bug in that bundle. Phase 5 skips blocked bundles.
3. **Surface to the user** with one line per blocked bundle: `⛔ fix-<bundle> blocked by worktree permission prompt for <tool> — user must grant or re-run with that tool pre-allowed`.

Do **not**:
- `cd` into the main repo to apply the fix.
- Use the orchestrator's own tool permissions to run the fix "just this once."
- Copy the worker's partial work into the main repo to finish it.

Blocked fixes are a clean failure. Fixes that silently leak onto main are a catastrophic failure. Always prefer the former.

### Post-implementer commit verification

After each implementer returns, before accepting its fix reports, the orchestrator verifies:

```bash
# Each reported commit_sha must exist on the bundle branch, NOT on main
for sha in $(jq -r '.commit_sha' "$STATE_DIR"/fixes/BUG-*.json); do
    ON_BUNDLE=$(git branch --contains "$sha" 2>/dev/null | grep -c "bounty/<run_id>/<bundle>" || echo 0)
    ON_MAIN=$(git branch --contains "$sha" 2>/dev/null | grep -c "^\s*main$" || echo 0)
    [ "$ON_BUNDLE" -eq 1 ] && [ "$ON_MAIN" -eq 0 ] || echo "LEAK $sha"
done
```

Any commit that lands on `main` (or `MAIN_START_BRANCH`) is a leak. Stop the run, recover the commit onto a dedicated `bounty-recovered-<run_id>-<bundle>` branch via `git reset --keep` + `git cherry-pick`, and restore main to `MAIN_START_HEAD`. Report to the user.

### Hard rules for every worker prompt

Every planner, implementer, and reviewer prompt must include this block verbatim:

```
You are in an isolated worktree on branch bounty/<RUN_ID>/<BUNDLE>.
- Do NOT `git switch`, `git checkout`, or `git branch -D` to any other branch.
- Do NOT push, do NOT open PRs. The orchestrator handles all network operations.
- Do NOT write files outside this worktree. The only path outside the worktree you may write is $STATE_DIR, and only the specific filenames named in your prompt.
- If you finish with uncommitted changes, commit them on this branch before returning.
```

### Pre-dispatch guard

Before dispatching any wave of workers, the orchestrator verifies:

1. Each worktree's branch is `bounty/<run_id>/<bundle>` (not an old name from a previous run).
2. Each worktree's branch base is `origin/main` (verify with `git merge-base --is-ancestor origin/main bounty/<run_id>/<bundle>`).
3. The main repo's HEAD is still `$MAIN_START_HEAD` and branch is still `$MAIN_START_BRANCH` (recorded in Phase 0c).

Any mismatch aborts the wave and triggers the leak sweep below.

### Post-phase leak sweep

After each phase that dispatches workers (Phase 4 planners, implementers, reviewers, and Phase 5 PR-builders), the orchestrator re-runs the checks from 0c:

```bash
CUR_HEAD="$(git -C "$MAIN_REPO" rev-parse HEAD)"
CUR_BRANCH="$(git -C "$MAIN_REPO" branch --show-current)"
CUR_DIRTY="$(git -C "$MAIN_REPO" status --porcelain | grep -vE '^\?\? \.temp/bounty' || true)"
```

If `CUR_HEAD != MAIN_START_HEAD` or `CUR_BRANCH != MAIN_START_BRANCH`: a subagent committed to the main repo. STOP. Report to the user. Do **not** auto-revert — the user decides.

If new files appeared under `CUR_DIRTY` that match bundle target files (i.e. the subagent wrote into the main repo instead of its worktree): move them to a dedicated `bounty-leak-<run_id>` branch with `git stash push --include-untracked …` then recover onto the branch (same pattern as `moo-cleanup`'s WIP branch handling). Log a loud warning and continue; the user will see the leak branch in `/bounty-cleanup`.

Print a one-liner at every sweep: `🧹 leak sweep @ <phase>: main unchanged, 0 leaks` (or the leak details).

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
3. **Re-render the running leaderboard on every state change AND at the start of every phase.** Triggers: new confirmation, new FP, new fix-approval, *and* the first message of each Phase 1–6 (even when nothing has scored yet). A long silent hunt phase is exactly when the user most needs to see the table. Use the exact format from Phase 0e — header, box-drawing characters, totals row — prefixed `🏆 BOUNTY CHAMPIONSHIP — standings @ phase N (run <RUN_ID>)`. Never substitute a text summary ("current scores: security 2, concurrency 1…") for the table; the table is the contract. **Also never substitute a `Write(leaderboard.json)` tool-preview for the table** — the JSON preview is implementation noise, the ASCII table is the scoreboard. Always emit the table as assistant stdout text right after any `leaderboard.json` update, so the table is the last thing the user sees before the next tool call. **Render Clawd (0f) above the table** with the frame matching the most recent event (confirmation, FP, fix-approved, lead-change, or the quiet-tick frame when the re-render was triggered by a phase boundary with no scoring change). Lead-change detection: diff the `#1` row against the previous re-render — a new specialist on top gets the crown frame.
4. **Never batch.** If three claims land during one poll tick, print three lines — do not summarize as "3 new claims."
5. Keep the poll loop bounded: exit as soon as `TaskList` shows all dispatched agents in that phase complete, or after the iteration cap (then fall through to the phase's final wait).

## Phase 1: Shared recon

Spawn **1 recon agent** in an isolated worktree (`isolation: "worktree"`, `model: "sonnet"`, `run_in_background: false`). Its job: produce a shared briefing that every hunter receives as cached context.

**Scale the time budget by scope size**:

```
files_in_scope = count of non-vendor files under --scope
budget_minutes = max(5, ceil(files_in_scope / 100))
```

So a 120-file module gets 5 minutes, a 500-file module gets 5 minutes, a 1200-file repo gets 12, a 3000-file monorepo gets 30. Record the chosen budget in `$STATE_DIR/config.json` as `recon_budget_minutes`.

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
- **The dismissed list** (contents of `$STATE_DIR/dismissed.yml` — see Phase 0d) inlined verbatim. Instruct the hunter: "Before filing a claim, scan this list; if the claim matches an entry's file-glob *and* category, do not file it unless you have strong evidence the rationale does not apply (e.g. the code has changed). If you override a dismissal, include `override_reason` in the claim JSON." Skip this entirely when the list is empty or `--ignore-dismissed` was set.
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
2. Every ~25s run a Bash call like `ls -t "$STATE_DIR"/claims/*.json 2>/dev/null` and diff against the printed set.
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

### Pre-filter against the dismissed list

Before dispatching any voters, walk every surviving claim and test it against the dismissed entries loaded at Phase 0d. A claim matches when both its file matches an entry's glob AND its category overlaps an entry's category token.

For each matching claim:

1. Write a synthetic `FALSE_POSITIVE` tally straight into `$STATE_DIR/tally.json` with `resolution_source: "dismissed-list"` and `dismissed_entry_id: "<id>"`.
2. Do **not** dispatch voters on that claim. It skips Phase 3 voting entirely.
3. Do **not** apply the −3 finder penalty — the hunter was given the list, the override may have been legitimate, and policy calls should not tank a hunter's score. Record `points_awarded: 0` with reason `dismissed-policy`.
4. Log one-liner:

```
📒 auto-FP: BUG-security-1-003  matches MOO-001 (Raw HTML in Moogento hint templates)
```

5. Re-render the leaderboard (totals increment under FP but score unchanged).

Skip this step entirely when the dismissed list is empty or `--ignore-dismissed` was set.

### Persist new false positives at end of Phase 3

After all votes have resolved and the leaderboard updates have settled, batch-prompt the user for every `FALSE_POSITIVE` whose `resolution_source` is **`voter`** (not `dismissed-list` — those are already in the file):

```
📒 3 new false positives surfaced this run. Persist to .bounty/dismissed.yml so future runs skip them?
   [1] BUG-security-1-007  app/Controller/Orders.php:142  "SQL injection via X-Forwarded-For"  — voters: 5 FP / 0 VALID
   [2] BUG-authz-1-004     app/Controller/Admin.php:88    "Missing ACL on admin action"         — voters: 3 FP / 1 VALID / 1 abstain
   [3] BUG-xss-1-002       app/View/helper.phtml:33       "Raw output in helper"                — voters: 4 FP / 0 VALID
   Reply:  (a) all  (b) pick 1,3  (c) none  (d) edit — default (a) in 30s
```

On `all` / timeout: append one entry per FP with a best-effort `title` (claim category + short file path) and `rationale` composed from the top voter reasons. On `pick`: keep only chosen ids. On `edit`: open `$EDITOR` on the generated YAML so the user can refine each `rationale` before committing. On `none`: skip.

Write to the live `.bounty/dismissed.yml` (not `$STATE_DIR/dismissed.yml`) so the update persists for future runs. Use atomic temp-file + rename:

```bash
tmp="$(mktemp "$MAIN_REPO/.bounty/dismissed.yml.XXXX")"
yq '.dismissed += [$new_entries]' "$MAIN_REPO/.bounty/dismissed.yml" --argjson new_entries "$ENTRIES_JSON" > "$tmp" \
  && mv "$tmp" "$MAIN_REPO/.bounty/dismissed.yml"
```

Skip this step when `persist_dismissed: false` was resolved at Phase 0d. Skip also when zero voter-confirmed FPs surfaced this run.

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
2. Persist the fallback in `$STATE_DIR/config.json` as `model_vote_effective: "sonnet"` so resumed runs don't re-probe
3. Retry the failed voter with sonnet
4. Keep going — do not hard-fail the phase on a voter-tier rate limit

### Live tailing during voting

Between dispatching a batch and it finishing, alternate ~20s poll cycles:

1. For each `<BUG-ID>.jsonl` in `$STATE_DIR/votes/`, compare its current line count to the count printed last cycle. Print the **new-vote one-liner** for each newly appended line.
2. When a claim's vote count reaches N−1 (all voters in), apply the resolution rules immediately and print the **claim-resolved one-liner** plus an updated leaderboard table (see Live progress output rules).
3. Break when every claim in the batch is resolved, then proceed to the next batch.

### Resolution rules

Read the tallied votes and apply:

| Outcome | Condition | Points |
|---------|-----------|--------|
| CONFIRMED | majority VALID among non-abstaining voters | finder **+1** |
| FALSE_POSITIVE | ≥⌈(N−1)/3⌉ FALSE_POSITIVE votes **and** majority is FALSE_POSITIVE | finder **−3** |
| INCONCLUSIVE | neither | 0 |

Update `$STATE_DIR/leaderboard.json` atomically — **Bash + minified JSON via `jq -cn`, not the `Write` tool** (see Phase 0d rationale). Then emit the ASCII championship table as assistant stdout text so the user sees the scoreboard, not the JSON file-write preview. Also update `$STATE_DIR/README.md` with the current leaderboard (see `references/scoring-rules.md` for format).

## Phase 3.5: Proof-of-concept (high/critical confirmed only)

Skip entirely when `--poc off` or `--no-fix` is set (no PRs will exist, no need to harden evidence).

The PoC phase is the last filter before Opus spend. A CONFIRMED vote on a SQL-injection claim is the voters' best judgement from reading the code; a successful exploit script is the voters' judgement plus a trigger that actually trips the bug. Phase 3.5 promotes claims that reproduce and demotes those that don't.

### Eligibility

| `--poc` value | Which claims get a PoC |
|---------------|------------------------|
| `auto` (default) | Every CONFIRMED claim with `severity ∈ {high, critical}` |
| `always` | Every CONFIRMED claim regardless of severity |
| `off` | None — phase is skipped |

`critical` claims that are already headed for a GitHub issue (not auto-fix, per Phase 4 severity policy) still get a PoC — the issue body will include the reproduction, which is exactly when a concrete PoC is most valuable.

### Worktree contract for PoC agents

PoC agents are uniquely dangerous: their job is to run code that demonstrates a vulnerability. A naive PoC that commits an exploit script into the bundle branch would land the exploit in the shipped PR. The worktree contract for Phase 3.5 is therefore **stricter** than the Phase 4 planner/implementer contract:

```
You are running a PoC for BUG-<id> in an isolated read-only worktree.
- Do NOT write any file under the worktree (the repo checkout). Treat it as read-only.
- Do NOT run `git add`, `git commit`, `git switch`, `git checkout`, or any branch operation.
- Do NOT push anything anywhere. You have no network.
- The ONLY paths you may write to are under $POC_DIR ($STATE_DIR/pocs/<BUG-ID>/) and
  the single file $STATE_DIR/pocs/<BUG-ID>.json.
- If reproducing requires running the application, invoke it from $POC_DIR with
  cwd=$POC_DIR — never from the worktree root. Copy (don't symlink) any sample
  inputs you need into $POC_DIR first.
```

The orchestrator enforces the contract: after a PoC agent returns, it runs `git -C <worktree> status --porcelain` and if anything is dirty, the PoC is rejected (`status: "contract_violation"`), the claim stays CONFIRMED (no demotion, no promotion), and a loud warning goes into the final report. The worktree is discarded and a replacement is cut from `origin/main` before the next phase.

### Dispatch

For each eligible CONFIRMED claim:

- `isolation: "worktree"` — one read-only worktree per PoC, off `origin/main`
- `model: "sonnet"` (same tier as the hunt — the PoC is a specialized re-read, not a code-write)
- `name: "poc-<BUG-ID>"`
- `run_in_background: true`
- prompt receives: the full claim JSON, the cited file + ±20 lines of context, the recon report, the repo's run/validation commands (from `CLAUDE.md` / `AGENTS.md`), `$POC_DIR`, `$STATE_DIR`, and the hard-rules block above verbatim

Dispatch all eligible PoCs in a single message so they run in parallel. Never block on Phase 3.5 — cap the wall time at 10 minutes per PoC and dispatch in batches of 8 to keep model pressure reasonable.

### Output contract

Each PoC writes exactly two paths:

1. `$STATE_DIR/pocs/<BUG-ID>/` — the exploit artifacts (script, sample payload, captured output, screenshots-as-text). **These never enter git.** `$STATE_DIR` lives under `.temp/bounty/` which is already ignored by the repo's `.gitignore`, but the hard-rules block above makes it explicit.
2. `$STATE_DIR/pocs/<BUG-ID>.json` — the structured verdict:

```json
{
  "bug_id": "BUG-security-1-001",
  "reproduced": true,
  "method": "curl -X POST .../orders -d \"ip=' OR 1=1 --\" → 200 with full log dump",
  "artifacts_dir": "/.../.temp/bounty/<run_id>/pocs/BUG-security-1-001/",
  "observed": "Returned all 12,482 rows from moo_nmsp_log",
  "expected_after_fix": "Returns 400 (prepared-statement binding rejects the quoted payload)",
  "elapsed_seconds": 47,
  "model": "sonnet",
  "at": "2026-04-19T14:55:02Z"
}
```

Required fields: `bug_id`, `reproduced`, `method`, `observed`, `expected_after_fix`. Missing any → the orchestrator marks the PoC `status: "malformed"` and treats the claim as if `--poc off` for this bug (keep CONFIRMED, no demotion).

### Resolution

Read each `pocs/<BUG-ID>.json` as it lands:

| PoC outcome | Effect on the claim |
|-------------|---------------------|
| `reproduced: true` | Claim keeps CONFIRMED. `poc: true` flag set on the claim for Phase 4 planner and PR body. Finder gets **+1 PoC bonus** (rewarded on the leaderboard under a new `poc_bonus` column) |
| `reproduced: false` | Claim is demoted to FALSE_POSITIVE. Finder gets the standard **−3**. The PoC JSON's `method` and `observed` become the rationale in the dismissed-list prompt at end of Phase 3 |
| `status: "contract_violation"` or `"malformed"` | Claim stays CONFIRMED, no bonus. Recorded in final report under "PoC contract violations" |

Update `$STATE_DIR/leaderboard.json` and re-render the championship table after each PoC resolves — PoC bonuses and demotions both move the standings.

### Live tailing

Between dispatch and the final wait, poll `$STATE_DIR/pocs/` on the same ~20–30s cycle used elsewhere and print a one-liner per new `<BUG-ID>.json`:

```
🧪 PoC/BUG-security-1-001  REPRODUCED  method=sqli-via-xff  finder +1 → 🛡️ Security
🧪 PoC/BUG-authz-1-007     DID-NOT-REPRODUCE  DEMOTED to FP  finder −3 → 🛡️ AuthZ
🧪 PoC/BUG-resources-1-002 CONTRACT VIOLATION (wrote to worktree) — quarantined
```

### Dismissed-file interaction

At end of Phase 3 the orchestrator already offers to append voter-confirmed false positives to `.bounty/dismissed.yml`. After Phase 3.5, **PoC-demoted** false positives are folded into that same prompt with a clear `[poc-demoted]` tag so the user can see which dismissals came from code-reading voters vs. from an exploit that failed to reproduce. The PoC's `method` / `observed` / `expected_after_fix` become the `rationale` for that dismissed-list entry.

## Phase 4: Bundled fixes (skip if `--no-fix`)

Confirmed bugs enter `$STATE_DIR/queue.json` with their severity. Instead of dispatching one fixer per bug (which scales poorly when 30+ bugs land), Phase 4 runs a **plan → implement → review** pipeline over **bundles** of related bugs.

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

Cluster fixable confirmed bugs into **bundles** and write `$STATE_DIR/bundles.json`. Default heuristic:

1. Start with one bundle per specialist lens that has ≥2 confirmed bugs.
2. Merge lenses with a single confirmed bug into a `misc` bundle for adjacent lenses (null-safety + error-handling often share one).
3. Cap at `--max-bundles` (default 6). When exceeded, merge the smallest bundles first.
4. Name each bundle with a kebab-slug describing the theme (e.g. `fix-acl`, `fix-fail-open`, `fix-concurrency`, `fix-nullsafety-misc`, `fix-data-integrity`, `fix-resources-perf`).

**Integrity check before proceeding.** After writing `bundles.json`, verify every `bug_id` in every bundle exists as a file under `$STATE_DIR/claims/`. If any referenced id is missing, the state is polluted (most often from a mid-run `/bounty` restart with stale `bundles.json` from a prior run). Abort Phase 4 with a clear message naming the missing ids and recommending `/bounty --fresh` or `/bounty-cleanup`. This is the exact desync seen in the April 2026 Smartcart run where `bundles.json` listed `BUG-authz-*` ids that were never in `claims/`.

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
  - Produce a markdown plan at `$STATE_DIR/plans/<bundle>.md` with YAML frontmatter:
    ```
    ---
    bundle: fix-acl
    test_strategy: tdd          # tdd | architectural — picks reviewer model (haiku vs. sonnet)
    commit_order: [BUG-authz-001, BUG-authz-002, BUG-authz-004]
    ---
    ```
  - One section per bug: proposed minimal fix, test stance (failing test that gets added, or why TDD won't work), and any dependencies on other bugs in the bundle
  - **No code changes.** Planning only. Read-only worktree contract: if the planner edits any file, its plan is rejected and the bundle falls back to per-bug fixing.

Poll `$STATE_DIR/plans/` for new files between dispatch and the final `TaskList` wait; print a plan-written one-liner when each lands:

```
📋 plans/fix-acl.md written  test_strategy=tdd  3 bugs  reviewer=haiku
```

### Step 4c: Implement (1 sonnet implementer per bundle)

After all plans are written, dispatch implementers in a single message:
- `isolation: "worktree"` — one worktree per bundle, branch `bounty/<run_id>/<bundle>` off `origin/main` (not the user's local `main` — `origin/main` is the canonical base so a stale local `main` does not contaminate the bundle)
- `model: "sonnet"` (or `$MODEL_FIX`)
- `name: "implementer-<bundle>"`
- `run_in_background: true`

Each implementer prompt includes:
- The bundle's plan (`$STATE_DIR/plans/<bundle>.md`) — passed verbatim
- The bundle's bug claims
- The recon report
- Project validation commands (from `CLAUDE.md` / `AGENTS.md`)
- Instructions to:
  - Walk the plan bug-by-bug in the order declared by `commit_order`
  - **TDD cycle per bug** when `test_strategy: tdd`: write the failing test first, confirm it fails, implement the minimal fix, confirm it passes, run validation, commit
  - One commit per bug. Never squash. Commit message follows the repo's `<module>: <benefit>` format and the three hard title rules in `references/scoring-rules.md#commit-message-rules`: (1) no "bounty" or tool branding, (2) no PR number (PR numbers belong in the PR body, not the commit title), (3) positive framing — no `fix`, `fixed`, `bug`, `bugs`, `error`, `issue`, `broken`, etc. in the title. Reference `BUG-<id>` in the body, never in the title.
  - Run project validation after each bug. If a bug cannot pass, skip it and write `$STATE_DIR/fixes/<BUG-ID>.json` with `{"status": "skipped", "reason": "..."}` — do not block the rest of the bundle
  - Write one `$STATE_DIR/fixes/<BUG-ID>.json` per bug on completion (files changed, test added, validation output, commit SHA)
  - Return a final bundle summary: `<bundle>: <done>/<total> fixed, <skipped> skipped`

Tail `$STATE_DIR/fixes/` on the same ~30s cycle and print the fix-committed one-liner per new report.

### Step 4d: Bundle review (1 reviewer per bundle)

After an implementer finishes its bundle, dispatch **1 reviewer**:
- `model: "haiku"` when the plan's frontmatter says `test_strategy: tdd` (the passing tests are most of the evidence)
- `model: "sonnet"` when `test_strategy: architectural` (review needs judgment)
- `name: "reviewer-<bundle>"`
- `isolation: "worktree"` (checks out `bounty/<run_id>/<bundle>`)
- Reviewer may not be the planner or implementer of this bundle. Prefer a specialist whose lens is adjacent but not represented among the bundle's original finders.

**The reviewer is blind.** The reviewer's prompt contains only:

1. The bug claim JSON(s) in the bundle — they need to know which bugs the diff is supposed to fix
2. The per-commit diff (`git log -p origin/main..bounty/<run_id>/<bundle>`) and per-commit messages
3. Project rules (`CLAUDE.md` / `AGENTS.md`) — the conventions they're measuring against
4. PoC evidence where available — `$STATE_DIR/pocs/<BUG-ID>.json`'s `method`, `observed`, `expected_after_fix` for any bug with a reproduced PoC

And explicitly **not**:

- The plan (`$STATE_DIR/plans/<bundle>.md`) — the reviewer must judge the diff on its own merits, not against the planner's narrative
- The implementer's `$STATE_DIR/fixes/<BUG-ID>.json` summaries — the commit + test + validation output should speak for the fix; a fix that needs the implementer's rationale to look correct is under-commented, and that's exactly what Step 4d should catch
- Any natural-language summary of what the bundle was trying to achieve, beyond the per-commit messages themselves

Rationale: borrowed from `wrxck/auto-audit`'s independent-review model. A reviewer who reads the planner's "why" first will rationalize a weak diff into an acceptable one ("the planner said this was fine, so the sparse test must be enough"). A blind reviewer either sees the bug and the fix or doesn't — and when they don't, the commit gets rejected and the bug returns to the queue, which is the correct outcome.

The reviewer inspects each per-bug commit on the bundle branch and returns one verdict per bug: `APPROVE` or `REJECT` with a one-sentence reason. Path: `$STATE_DIR/reviews/<bundle>.json`.

Resolution per bug:
- `APPROVE` → fix kept on the bundle branch, implementer gets **+1**
- `REJECT` → revert that single commit on the bundle branch (`git revert <sha>`); the bug returns to the queue for **one** isolated retry by a different agent (per-bug fix, opus), then abandoned if still rejected

Print the bundle-review one-liner per bug and re-render the leaderboard after each change.

## Phase 5: Ship

A single PR with 30+ bugs and 1,500+ changed lines is unreviewable. Phase 5 therefore splits approved commits across **multiple PRs**, each sized for a realistic human review. One mega-PR and a "document and leave for later" pile are both wrong answers — every approved fix gets shipped in a PR of its own group.

### Limits

| Threshold | Default | Override |
|-----------|---------|----------|
| Max bugs per PR | 10 | `--max-bugs-per-pr` |
| Max added+removed lines per PR | 400 | `--max-lines-per-pr` |

Either threshold triggers a split. Exception: a single commit that alone exceeds `--max-lines-per-pr` gets its own PR — the policy never splits an individual fix across PRs. Mark those oversize PRs in the title (`<module>: large fix — <theme>`) so reviewers know what they're looking at.

### Step 5a: Group approved commits into PR batches

Build `$STATE_DIR/pr-groups.json`. Algorithm:

1. Start with the list of approved per-bug commits across all bundle branches, preserving the bundle's `commit_order`.
2. Group greedily by bundle theme first (bundles are cohesive by construction), then by severity (high-severity bugs batch together for faster review).
3. For each group, measure `bug_count` and `est_lines = Σ (additions + deletions)` per commit via `git log --numstat bundle-branch..origin/main`.
4. When adding the next commit would exceed **either** threshold, close the current group and open a new one. Keep all of a bundle's commits in the same group when they fit; only split a bundle across PRs when the bundle itself exceeds a threshold.
5. Name each group `bounty/<run_id>/pr-<N>-<slug>` where `<slug>` is the dominant theme's kebab-name.

Write `pr-groups.json`:

```json
{
  "run_id": "20260420-143000-smartcart",
  "groups": [
    {"slug": "bounty/20260420-143000-smartcart/pr-1-acl-and-csrf", "bundles": ["fix-acl", "fix-csrf"], "bug_ids": [...], "commits": [...], "bug_count": 8, "est_lines": 312},
    {"slug": "bounty/20260420-143000-smartcart/pr-2-null-safety",  "bundles": ["fix-null-safety"],  "bug_ids": [...], "commits": [...], "bug_count": 7, "est_lines": 268},
    ...
  ]
}
```

Print the plan as a one-liner per group so the user sees it before any branch is pushed:

```
📦 PR 1/3  pr-1-acl-and-csrf      8 bugs  312 lines  [fix-acl + fix-csrf]
📦 PR 2/3  pr-2-null-safety       7 bugs  268 lines  [fix-null-safety-bulk]
📦 PR 3/3  pr-3-concurrency       5 bugs  391 lines  [fix-concurrency]
```

### Step 5b: Build each aggregate branch

For each group:

1. Create `<slug>` off `origin/main` in an isolated worktree (reuses the same `isolation: "worktree"` pattern as Phase 4).
2. Cherry-pick the approved per-bug commits from their bundle branches, preserving `commit_order`. Never squash — the individual commits are the audit trail.
3. Run project validation on the aggregate branch. If it fails, bisect, drop the offending commit, and record the drop in `pr-groups.json` so the final report names it.
4. Re-measure `est_lines` post-validation; if a commit was dropped, the group's line count may now fit under the threshold and the next group's first commit can slide back in. Re-balance once. Do not re-balance twice — infinite rebalancing masks real problems.

Emit a leak-sweep check (see Branch isolation contract) after all aggregates are built.

### Step 5b.5: Pre-push secrets scan

Skip when `--no-secrets-scan` is set. Default: on.

Before any aggregate branch is pushed, scan its diff against `origin/main` for credential patterns. A confirmed bug fix that accidentally commits a `.env` value, a test-fixture API key, or a snippet of log output containing a bearer token is a much worse outcome than the bug it was trying to fix. The pre-push secrets scan is the last gate where that commit can be stopped.

For each aggregate branch in `pr-groups.json`:

```bash
DIFF="$(git -C <worktree> diff origin/main..HEAD -- . ':(exclude)*.lock' ':(exclude)*.min.js' ':(exclude)*.min.css')"
```

Run the diff through a pattern sweep. The orchestrator is allowed a small fixed pattern set (not the LLM's judgement) so this step runs the same way every time:

| Pattern | What it catches |
|---------|-----------------|
| `(?i)(aws|amazon).{0,20}(access|secret)[_-]?key` followed by `= *["']?[A-Z0-9/+]{20,}` | AWS access/secret keys |
| `AKIA[0-9A-Z]{16}` | AWS access key id |
| `xox[baprs]-[0-9a-zA-Z]{10,}` | Slack tokens |
| `ghp_[0-9A-Za-z]{36}`, `gho_[0-9A-Za-z]{36}`, `ghs_[0-9A-Za-z]{36}`, `github_pat_[0-9A-Za-z_]{22,}` | GitHub tokens |
| `-----BEGIN (RSA |EC |OPENSSH |DSA |PGP )?PRIVATE KEY-----` | Private keys |
| `(?i)(api[_-]?key|bearer|authorization|token|password|passwd|secret|credential)[[:space:]]*[:=][[:space:]]*["'][A-Za-z0-9_\-\./+=]{16,}["']` | Generic high-entropy token assignments |
| `eyJ[A-Za-z0-9_\-]{20,}\.[A-Za-z0-9_\-]{20,}\.[A-Za-z0-9_\-]{20,}` | JWT tokens |

Use `git grep -nE` against the diff (staged via a temp branch) or pipe the `git diff` output through `grep -nE` directly. The exact pattern file lives at `skills/bounty/references/secret-patterns.txt` and is read at Phase 0 so the run uses a stable snapshot.

Any hit aborts the push for **that PR group only** (other groups still ship if they're clean). Record the hit in `$STATE_DIR/pr-groups.json` under the group's `blocked_by_secrets_scan` field:

```json
{
  "slug": "bounty/20260420-143000-smartcart/pr-2-null-safety",
  "status": "blocked",
  "blocked_by_secrets_scan": [
    {"commit": "a1b2c3d", "file": "tests/fixtures/config.php", "line": 17, "pattern": "aws-secret-key", "match": "AWS_SECRET_KEY = \"wJa…redacted…\""}
  ]
}
```

The final report surfaces blocked groups at the top (not buried with the leak-sweep log) so the user can reconcile whether the match is a true secret or a test-fixture false positive. The matched secret itself is **never** written to the final report or the leaderboard — only the pattern name, file, and line. This matters because `.temp/bounty/` is sometimes shared via screenshot for debugging.

Print a one-liner per blocked group:

```
🚫 PR 2/3  pr-2-null-safety  BLOCKED by secrets-scan  tests/fixtures/config.php:17 (aws-secret-key)
```

On a hit the finder of the underlying bug is **not** penalized — the fix that leaked the secret was the implementer's work, not the hunter's. Instead, the implementer loses their `+1` for that bug (the approval is reversed on the secrets-scan hit) and the bug returns to the queue for one isolated per-bug retry. If the retry also hits the scan, abandon the bug and flag it in the final report.

### Step 5c: Push and open PRs

Push each aggregate branch and open its PR via `gh pr create`. Use the template from `references/scoring-rules.md`.

**PR titles must read like normal product work.** Lead with the module, followed by a positive benefit statement (imperative-past). Do NOT mention "bounty", "bounty sweep", "bounty run", or numeric bug counts in the title — those live in the body. See `references/scoring-rules.md#title-style--positive-module-generic-no-bounty-branding` for worked examples. Summary: `smartcart: Reduced risk and improved reliability` — not `smartcart: Resolved 11 issues from bounty sweep`.

The same rule applies to any `gh pr merge --subject …` override the orchestrator might generate when merging (e.g. when `/bounty-cleanup` is configured to squash) — the squash subject should mirror the PR title style, never reintroduce bounty branding.

Cross-link sibling PRs in every body:

```
> Part N of M from bounty run 20260420-143000-smartcart.
> Siblings: #<pr-1>, #<pr-2>, #<pr-3>.
```

Record every PR url in `$STATE_DIR/pr-groups.json` under each group's `pr_url` and in `$STATE_DIR/README.md`.

### Step 5d: Critical issues

For each `critical` bug, open a separate GitHub issue via `gh issue create` instead of committing. Link the issue from the final report.

### Step 5e: Automated PR review feedback loop

Modeled on `/moo`'s `moo-pr-feedback-handler` — after every PR lands in Step 5c, CI runs and automated reviewers (GitHub Advanced Security, CodeRabbit, Copilot review, the Claude review bot, etc.) typically post within 5–15 minutes. Step 5e reads their comments, validates them, fixes the valid ones in-place on the PR branch, and replies with a rationale to any that are wrong. Loop up to `--pr-review-rounds` (default 2) per PR.

Skip Step 5e entirely when `--no-fix` was passed (no PRs exist) or when Step 5c opened zero PRs (e.g. every commit was dropped during validation).

All state for this step lives under `$STATE_DIR/pr-review/<pr-number>.json`:

```json
{
  "pr_number": 1234,
  "branch": "bounty/20260420-143000-smartcart/pr-1-acl-and-csrf",
  "rounds": [
    {
      "round": 1,
      "wait_seconds": 300,
      "checks_at_start": {"passed": 3, "failed": 0, "pending": 2},
      "processed_comment_ids": [892341, 892352, 892399],
      "evaluated": [
        {"comment_id": 892341, "user": "coderabbitai[bot]", "path": "app/Http/Orders.php", "line": 142, "verdict": "VALID", "reason": "Genuine SQL concat bypass; add prepared statement."},
        {"comment_id": 892352, "user": "coderabbitai[bot]", "path": "app/Http/Cart.php",   "line":  63, "verdict": "INVALID", "reason": "Suggested caching breaks live-pricing contract — documented in ADR-017."},
        {"comment_id": 892399, "user": "github-actions[bot]", "path": null, "line": null, "verdict": "SKIP", "reason": "Informational summary only."}
      ],
      "fixes_committed": ["a1b2c3d"],
      "rejection_comment_id": 892410,
      "pushed": true,
      "finished_at": "2026-04-20T04:12:00Z"
    }
  ]
}
```

**Dispatch in parallel across PRs.** All opened PRs can run their feedback loops concurrently — each PR has its own worktree and its own state file. Send one `bounty-pr-review-<N>` worker per PR in a single message.

Each worker's prompt includes `PR_NUMBER`, `PR_BRANCH`, the PR worktree path, `STATE_DIR`, `MAX_ROUNDS=--pr-review-rounds`, `WAIT_SECONDS=--pr-review-wait`, and the repo's validation commands (from `CLAUDE.md` / `AGENTS.md`, cached in `config.json`). The worker then runs this loop (rounds 1…MAX_ROUNDS), mirroring `/moo`'s `moo-pr-feedback-handler`:

**1. Wait for checks.** Poll `gh pr checks $PR_NUMBER` every 60s up to `WAIT_SECONDS`, breaking early when no check is `pending / in_progress / queued / running`. Report check failures (they are *not* the same as review comments — fix-attempts happen regardless, but failed checks go in the final report).

**2. Read comments.** Both the issue-comments and review-comments endpoints:

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
gh api "repos/$REPO/issues/$PR_NUMBER/comments" \
  --jq '.[] | {id, user: .user.login, body, created_at}'
gh api "repos/$REPO/pulls/$PR_NUMBER/comments" \
  --jq '.[] | {id, user: .user.login, body, path, line, created_at}'
```

Drop every id already present in `processed_comment_ids` from previous rounds. If zero new actionable comments, exit the loop early and record `{"exit_reason": "no_new_feedback"}`.

**3. Evaluate each suggestion.** Read the referenced file and ±10 lines of context before judging. Label every comment:

| Verdict | When |
|---------|------|
| `VALID` | Real issue — bug, security hole, missing test, style violation the repo enforces |
| `INVALID` | Wrong — misreads the code, conflicts with a documented ADR/comment, would break behavior |
| `SKIP` | Informational / praise / not actionable |

Never blind-apply. A suggestion that's technically correct but conflicts with an intentional design choice is `INVALID` with a pointer to the ADR, comment, or test that encodes the intent.

**4. Implement valid fixes in-place** on the PR's branch (worker is already on it). Keep fixes minimal — do not refactor beyond what the comment asked for. After every fix, run the repo's validation (`vendor/bin/phpcs`, `phpunit`, `npm run lint`, etc. as configured). If validation fails, the fix is rolled back and recorded as `{"verdict": "VALID", "applied": false, "reason": "validation regressed"}` — the review comment gets a rationale reply instead.

**5. Reply with rationale for every INVALID** (one consolidated comment per round, only when `config.pr_review_replies == true` *and* at least one `INVALID` this round):

```bash
gh pr comment $PR_NUMBER --body "$(cat <<'EOF'
## Bounty automated-review response — round <N>

The following suggestions were evaluated by the bounty run and not applied:

- **<file>:<line>** — <what was suggested> — <why rejected>

These points have been reviewed and will not be addressed in this PR.
EOF
)"
```

Skip the reply comment entirely when every new comment was `VALID` or `SKIP`, **or** when `pr_review_replies` is `false` (the user opted out at Phase 0). Either way, the worker records the same INVALID verdicts + rationale in `$STATE_DIR/pr-review/<pr-number>.json` so the final report can surface them even when posting is off.

**6. Commit and push** fixes as a single combined commit per round. Follow the three hard title rules in `references/scoring-rules.md#commit-message-rules`: no "bounty" or tool branding, no PR number (not even as `(#1234)` suffix — the platform adds its own metadata), positive framing only. Lowercase module, no AI attribution, no `fix:` / `feat:` prefix:

```
<module>: improved code quality from automated review (round <N>)

Applied valid suggestions from the round-<N> automated PR review.

Changes:
- <specific fix 1>
- <specific fix 2>

PR #<pr-number>, bounty <run_id>
```

**7. Repeat** up to `MAX_ROUNDS`. Stop early when a round produces zero new actionable comments, or when every new comment was `INVALID` / `SKIP`. Append each round's record to the worker's `$STATE_DIR/pr-review/<pr-number>.json` atomically.

**Live output** per round, on stdout:

```
🤖 PR #1234 round 1: 7 new comments · VALID 4 · INVALID 2 · SKIP 1
✍  PR #1234 round 1: 4 fixes pushed (commit a1b2c3d), rejection reply posted for 2
```

When every PR's feedback loop has exited (naturally or at MAX_ROUNDS), proceed to Phase 6. The final report names every fix-round commit and every rejection reply.

Branch isolation contract still applies: the worker never switches branches, never pushes anything except to its own PR branch, never touches `main`, and never writes files outside its worktree or `$STATE_DIR`. Run the post-phase leak sweep when every worker returns.

## Phase 6: Final report

Render the final leaderboard to stdout and to `$STATE_DIR/README.md`:

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
- **Automated-review outcomes** per PR: rounds run, fixes applied in-round (with commit SHAs), rejections posted (with comment IDs). Pull from `$STATE_DIR/pr-review/<pr-number>.json`.
- Total wall clock and rough token cost estimate

## Cleanup

Run `/bounty-cleanup` when the run finishes. By default it scopes to **this run's `run_id`** — detected from the current branch if it's `bounty/<run_id>/…`, or from `.temp/bounty/latest` otherwise. It merges any open PRs whose head is `bounty/<run_id>/…` (same flow as `moo-cleanup` — per-PR leak sweep + WIP-branch preservation), fast-forwards the default branch, tears down worktrees and branches under `bounty/<run_id>/…`, and removes `$STATE_DIR` (i.e. `.temp/bounty/<run_id>/`). Other in-progress bounty runs are left alone.

Pass `--run-id <id>` to target a specific run, `--all` to clean every bounty artifact across all runs, `--keep-state` to preserve the audit trail, or `--no-merge` to skip the merge step.

## Rules — non-negotiable

1. Agents cannot vote on their own claims. The orchestrator enforces this by excluding the finder from that claim's voter set.
2. Agents cannot review fixes they wrote or claims they found.
3. Every claim needs a file path, line number, and evidence snippet. Claims missing any of these are rejected by the orchestrator before voting.
4. Follow the repo's commit rules (no AI attribution, no `fix:` / `feat:` prefixes, lowercase module prefix before colon).
5. Each run's state lives under its own `.temp/bounty/<run_id>/` directory (exposed to orchestrator and subagents as `$STATE_DIR`). Never mutate repo files outside fix worktrees. Never write into another run's state dir.
6. Every branch this run creates is `bounty/<run_id>/<bundle>` (or `bounty/<run_id>/pr-*`). Never commit to `main` or the user's starting branch. The orchestrator enforces this via the Branch isolation contract and runs a leak sweep after every worker phase.
7. `bundles.json` must only reference `bug_id`s that exist in `$STATE_DIR/claims/`. Any desync aborts Phase 4 with a clear recovery message.
8. **Never fall back to the main-repo checkout when a worktree worker is blocked.** A blocked worker yields a `status: "blocked"` fix report, not a shortcut that writes to main. Review the "fall back to main" anti-pattern in the Branch isolation contract.

## Additional Resources

- `references/agent-specialties.md` — the 8 specialist lenses with concrete patterns
- `references/scoring-rules.md` — scoring, thresholds, leaderboard and PR templates
- `references/state-schema.md` — JSON schemas for claims, votes, leaderboard, fixes
