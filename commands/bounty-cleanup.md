---
description: Merge open PRs from a bounty run and clean up its worktrees, branches, and state — scoped to one run by default
argument-hint: "[--run-id <id>] [--all] [--keep-state] [--no-merge]"
allowed-tools: Bash(git:*), Bash(gh:*), Bash(rm:*), Bash(ls:*), Bash(find:*), Bash(jq:*), Bash(readlink:*), Read
model: haiku
---

# Bounty Cleanup

Tear down artifacts from **one** bounty run. Other in-progress bounty runs in the same repo are left alone unless `--all` is given.

**Flags**:
- `--run-id <id>` — target this specific run
- `--all` — ignore run scoping and clean every bounty artifact across all runs
- `--keep-state` — preserve `.temp/bounty/<run_id>/` for audit
- `--no-merge` — skip the PR-merge step, just tear down

## Step 0: Resolve the target run id

Default behavior is **scoped**: clean only the run the user is in. Resolve `RUN_ID` in this order (stop at the first hit):

```bash
MAIN_REPO="$(git rev-parse --show-toplevel)"
cd "$MAIN_REPO"

# 1. Explicit flag wins
if [ -n "$ARG_RUN_ID" ]; then
    RUN_ID="$ARG_RUN_ID"

# 2. Current branch: bounty/<run-id>/<bundle>
elif [[ "$(git branch --show-current)" =~ ^bounty/([^/]+)/ ]]; then
    RUN_ID="${BASH_REMATCH[1]}"

# 3. Current worktree's branch (if we're inside a worktree)
elif [[ "$(git -C . symbolic-ref --short HEAD 2>/dev/null)" =~ ^bounty/([^/]+)/ ]]; then
    RUN_ID="${BASH_REMATCH[1]}"

# 4. .temp/bounty/latest symlink (points to the most recent run)
elif [ -L .temp/bounty/latest ]; then
    RUN_ID="$(readlink .temp/bounty/latest)"

# 5. Exactly one in-progress run under .temp/bounty/
elif [ "$(find .temp/bounty -mindepth 2 -maxdepth 2 -name config.json 2>/dev/null | wc -l)" = "1" ]; then
    RUN_ID="$(find .temp/bounty -mindepth 2 -maxdepth 2 -name config.json | xargs -n1 dirname | xargs -n1 basename)"
fi
```

If `--all` was given, set `RUN_ID=""` and skip the scoping checks — every `bounty/*` branch, worktree, PR, and state dir is in scope.

If no `RUN_ID` could be resolved and `--all` was not given: print the list of `.temp/bounty/<id>/` directories (if any) plus any `bounty/*` branches and ask the user to pick one with `--run-id` or confirm `--all`. Stop.

Announce the resolved scope before touching anything:

```
🎯 bounty-cleanup scoped to run_id = 20260420-143000-smartcart
   (use --all to clean every bounty artifact, or --run-id <id> to target a different run)
```

Compute the filter prefix used throughout the rest of the script:

```bash
if [ -n "$RUN_ID" ]; then
    BRANCH_FILTER="bounty/${RUN_ID}/"      # exact run prefix
    PR_SEARCH="head:bounty/${RUN_ID}/"
    STATE_TARGET=".temp/bounty/${RUN_ID}"
else
    BRANCH_FILTER="bounty/"                 # --all
    PR_SEARCH="head:bounty/"
    STATE_TARGET=".temp/bounty"
fi
```

## Step 1: Inspect the scoped state

```bash
git worktree list | grep -E "$BRANCH_FILTER" || true
git branch --list "${BRANCH_FILTER}*"
ls -la "$STATE_TARGET" 2>/dev/null || echo "(no state dir)"

OPEN_PRS=$(gh pr list --state open --search "$PR_SEARCH" --json number,title,headRefName,state --limit 100)
echo "$OPEN_PRS" | jq -r '.[] | "PR #\(.number)  \(.headRefName)  \(.title)"'
```

If `$STATE_TARGET/config.json` exists, surface its `run_id` and `last_completed_phase` so the user sees which run this cleans up.

## Step 2: Merge each open bounty PR (skip if `--no-merge`)

For each PR in `OPEN_PRS`, run the full moo-cleanup flow one PR at a time — merging in a batch without sweeping between each re-pollutes main after the first merge.

For each PR:

### 2a. Merge

```bash
PR_NUMBER=<from list>
BRANCH_NAME=<headRefName>

gh pr merge "$PR_NUMBER" --squash --delete-branch \
  || gh pr merge "$PR_NUMBER" --merge --delete-branch
```

**Do not pass `--subject` unless you have a specific reason.** Without it, GitHub uses the PR title as the squash subject, which is what we want. If you *must* override it, mirror the PR title style (`<module>: <positive benefit>`) — no "bounty", "bounty sweep", "bounty run", or numeric bug counts, no `fix:` / `feat:` prefixes. See `skills/bounty/references/scoring-rules.md#title-style--positive-module-generic-no-bounty-branding`. The title ends up in the repo's permanent git log; it should read like normal product work, not like tool output.

If the merge fails (checks required, conflicts, permission), report the error and skip this PR. Move to the next.

### 2b. Detect subagent leaks vs WIP

```bash
PR_FILES=$(gh pr view "$PR_NUMBER" --json files --jq '.files[].path' | sort -u)

# Scope LOCAL_CHANGES to exclude every other run's state dir, not just this one's
LOCAL_CHANGES=$(git status --porcelain \
    | awk '{ if ($1 == "??") print $2; else print $2 }' \
    | grep -vE '^(\.temp/bounty/|\.plans/|\.claude/settings\.local\.json$)' \
    | sort -u)

LEAKS=$(comm -12 <(echo "$PR_FILES") <(echo "$LOCAL_CHANGES"))
WIP=$(comm -13 <(echo "$PR_FILES") <(echo "$LOCAL_CHANGES"))
echo "PR #$PR_NUMBER — $(echo "$LEAKS" | wc -l | tr -d ' ') leaked file(s), $(echo "$WIP" | wc -l | tr -d ' ') WIP file(s)"
```

### 2c. Discard leaks

```bash
if [ -n "$LEAKS" ]; then
    echo "$LEAKS" | while IFS= read -r f; do
        [ -z "$f" ] && continue
        if git ls-files --error-unmatch "$f" &>/dev/null; then
            git checkout -- "$f"
        else
            rm -f "$f"
        fi
    done
fi
```

### 2d. Preserve WIP on a dedicated branch

```bash
if [ -n "$WIP" ]; then
    WIP_BRANCH="wip/bounty-${BRANCH_NAME##*/}-leftover-$(date +%Y%m%d)"
    git stash push --include-untracked -m "bounty-cleanup: staging WIP for $WIP_BRANCH"
    git checkout -b "$WIP_BRANCH"
    git stash pop
    git add -A
    git commit -m "wip: recovered leftover changes from $BRANCH_NAME cleanup" \
        -m "These files were in the main repo when $BRANCH_NAME was merged." \
        -m "Not part of the merged PR; may be WIP from a prior session."
    git checkout "$(git symbolic-ref refs/remotes/origin/HEAD | sed 's|refs/remotes/origin/||')"
fi
```

### 2e. Fast-forward the default branch

```bash
DEFAULT_BRANCH="$(git symbolic-ref refs/remotes/origin/HEAD | sed 's|refs/remotes/origin/||')"
git fetch origin "$DEFAULT_BRANCH"
git checkout "$DEFAULT_BRANCH" 2>/dev/null || true

UNPUSHED=$(git rev-list "origin/${DEFAULT_BRANCH}..${DEFAULT_BRANCH}" --count 2>/dev/null)
if [ "$UNPUSHED" -gt 0 ]; then
    git log --oneline "origin/${DEFAULT_BRANCH}..${DEFAULT_BRANCH}"
    git rebase "origin/${DEFAULT_BRANCH}" \
        || { echo "Rebase failed — aborting"; git rebase --abort; exit 1; }
else
    git merge --ff-only "origin/${DEFAULT_BRANCH}"
fi
```

Never `git reset --hard`. Never skip the unpushed-commit check.

## Step 3: Classify remaining bounty branches (in scope)

```bash
for b in $(git branch --list "${BRANCH_FILTER}*" --format='%(refname:short)'); do
  if git merge-base --is-ancestor "$b" "$DEFAULT_BRANCH" 2>/dev/null; then
    echo "MERGED  $b"
  else
    echo "UNMERGED $b"
  fi
done
```

Show both lists. Confirm before deleting any unmerged branch. Branches outside `$BRANCH_FILTER` are never touched — they belong to other in-progress runs or to the user.

## Step 4: Remove bounty worktrees (in scope)

```bash
for path in $(git worktree list --porcelain | awk '/^worktree / {print $2}'); do
    # Only remove worktrees whose branch matches the filter
    wt_branch=$(git -C "$path" branch --show-current 2>/dev/null)
    case "$wt_branch" in
        ${BRANCH_FILTER}*) git worktree remove "$path" --force ;;
    esac
done
git worktree prune
```

## Step 5: Delete merged branches (in scope)

```bash
for b in $(git branch --list "${BRANCH_FILTER}*" --format='%(refname:short)' --merged "$DEFAULT_BRANCH"); do
  git branch -d "$b"
done
```

Unmerged branches matching the filter: only `-D` after the user confirms. Branches outside the filter are left alone unconditionally.

## Step 6: Remove this run's state directory

Unless `--keep-state` was given:

```bash
rm -rf "$STATE_TARGET"

# If --all and the whole .temp/bounty/ is now empty, remove it
if [ -z "$RUN_ID" ] && [ -d .temp/bounty ] && [ -z "$(ls -A .temp/bounty 2>/dev/null)" ]; then
    rmdir .temp/bounty
fi

# Remove the latest symlink if it pointed to this run
if [ -n "$RUN_ID" ] && [ "$(readlink .temp/bounty/latest 2>/dev/null)" = "$RUN_ID" ]; then
    rm -f .temp/bounty/latest
fi

# Stray temp-file sweep (safe globally — these are always bounty-owned)
find . -maxdepth 2 -name '.temp.bounty*' -type f -delete 2>/dev/null
find . -maxdepth 2 -name 'bounty-*.tmp' -type f -delete 2>/dev/null
rm -f /tmp/bounty-*.tmp 2>/dev/null
```

If another in-progress run still has a state dir, leave `.temp/bounty/` itself in place.

## Step 7: Verify

```bash
git worktree list
git branch --list "${BRANCH_FILTER}*" | wc -l
gh pr list --state open --search "$PR_SEARCH" --json number | jq length
ls -la "$STATE_TARGET" 2>/dev/null || echo "$STATE_TARGET cleared"

# Surface any bounty state that was deliberately preserved
echo "--- other bounty runs still present ---"
ls -la .temp/bounty 2>/dev/null | grep -vE '^(total|\.\.?$|latest)' || echo "(none)"
```

Expected on a clean run (scoped): 0 in-scope worktrees, 0 in-scope branches, 0 in-scope open PRs, `$STATE_TARGET` cleared. Other runs still visible.

## Output

Report:
- Target run_id (or `--all`)
- PRs merged (numbers + titles)
- PR merge failures (if any) with the error
- Leak count per merged PR (files discarded)
- WIP branches created (if any) — prompt the user to review
- Worktrees removed
- Merged branches deleted
- Unmerged branches left (if any), with a reminder that they are the user's call
- State dir cleared or preserved (call out `plans/`, `fixes/`, `reviews/` explicitly)
- Other in-progress bounty runs still present (so the user knows they weren't touched)
- Any stray temp files removed or left behind

If any step fails, report what succeeded and what didn't so the user can finish manually.
