---
name: bounty-cleanup
description: Use when the user wants to merge an open bounty PR, remove bounty worktrees and `bounty/*` branches, or clear `.temp/bounty/<run_id>/` state from one run. Scopes to one run by default; leaves other in-progress runs alone.
metadata:
  short-description: Merge open bounty PRs and clean up one run's branches, worktrees, and state
---

# Bounty Cleanup

Tear down artifacts from **one** bounty run. Other in-progress bounty runs in the same repo are left alone unless `--all` is given. Adapted for Codex from the same flow the Claude plugin ships.

## Flags

- `--run-id <id>` — target this specific run
- `--all` — ignore run scoping and clean every bounty artifact across all runs
- `--keep-state` — preserve `.temp/bounty/<run_id>/` for audit
- `--no-merge` — skip the PR-merge step, just tear down

## Safety Rules

1. Delete merged `bounty/*` branches (within scope) without prompting.
2. Ask before deleting any unmerged branch within scope.
3. Never touch branches, worktrees, PRs, or state dirs that belong to a different run unless `--all`.
4. Preserve `.temp/bounty/<run_id>/` when `--keep-state`.
5. Never `git reset --hard`. Preserve both uncommitted and unpushed committed work.

## Step 0: Resolve the target run id

Default behavior is **scoped**: clean only the run the user is in. Resolve in order:

1. `--run-id <id>` argument — wins.
2. Current branch name matching `bounty/<id>/<bundle>` — `<id>` is the target.
3. Current worktree's branch if inside a worktree, same pattern.
4. `.temp/bounty/latest` symlink — points to the most recent run.
5. Exactly one in-progress run under `.temp/bounty/` — adopt it.

If `--all` is given, skip scoping entirely. Otherwise if no run id resolves, list the in-progress runs (and any stray `bounty/*` branches) and ask the user to pick `--run-id <id>` or confirm `--all`. Stop.

Announce the resolved scope:

```
🎯 bounty-cleanup scoped to run_id = 20260420-143000-smartcart
   (use --all to clean every bounty artifact)
```

Compute filters used throughout:

```bash
if [ -n "$RUN_ID" ]; then
    BRANCH_FILTER="bounty/${RUN_ID}/"
    PR_SEARCH="head:bounty/${RUN_ID}/"
    STATE_TARGET=".temp/bounty/${RUN_ID}"
else
    BRANCH_FILTER="bounty/"
    PR_SEARCH="head:bounty/"
    STATE_TARGET=".temp/bounty"
fi
```

## Step 1: Inspect the scoped state

```bash
git worktree list | grep -E "$BRANCH_FILTER" || true
git branch --list "${BRANCH_FILTER}*"
ls -la "$STATE_TARGET" 2>/dev/null || echo "(no state dir)"
gh pr list --state open --search "$PR_SEARCH" --json number,title,headRefName --limit 100
```

Surface the run's `run_id` and `last_completed_phase` from `$STATE_TARGET/config.json` so the user knows which run this targets.

## Step 2: Merge each open bounty PR (skip if `--no-merge`)

One PR at a time — merging in a batch without sweeping between each re-pollutes main after the first merge.

For each PR:

### 2a. Merge

```bash
gh pr merge "$PR_NUMBER" --squash --delete-branch \
  || gh pr merge "$PR_NUMBER" --merge --delete-branch
```

If merge fails (required checks, conflicts, permission), record the error and skip this PR.

### 2b. Detect subagent leaks vs WIP

After the PR lands, classify local main-repo changes:

```bash
PR_FILES=$(gh pr view "$PR_NUMBER" --json files --jq '.files[].path' | sort -u)
LOCAL_CHANGES=$(git status --porcelain \
    | awk '{ if ($1 == "??") print $2; else print $2 }' \
    | grep -vE '^(\.temp/bounty/|\.plans/|\.claude/settings\.local\.json$)' \
    | sort -u)
LEAKS=$(comm -12 <(echo "$PR_FILES") <(echo "$LOCAL_CHANGES"))
WIP=$(comm -13 <(echo "$PR_FILES") <(echo "$LOCAL_CHANGES"))
```

### 2c. Discard leaks

```bash
echo "$LEAKS" | while IFS= read -r f; do
    [ -z "$f" ] && continue
    if git ls-files --error-unmatch "$f" &>/dev/null; then
        git checkout -- "$f"
    else
        rm -f "$f"
    fi
done
```

### 2d. Preserve WIP on a dedicated branch

```bash
WIP_BRANCH="wip/bounty-${BRANCH_NAME##*/}-leftover-$(date +%Y%m%d)"
git stash push --include-untracked -m "bounty-cleanup: staging WIP for $WIP_BRANCH"
git checkout -b "$WIP_BRANCH"
git stash pop
git add -A
git commit -m "wip: recovered leftover changes from $BRANCH_NAME cleanup" \
    -m "Not part of the merged PR; may be WIP from a prior session."
git checkout "$(git symbolic-ref refs/remotes/origin/HEAD | sed 's|refs/remotes/origin/||')"
```

### 2e. Fast-forward the default branch

```bash
DEFAULT_BRANCH="$(git symbolic-ref refs/remotes/origin/HEAD | sed 's|refs/remotes/origin/||')"
git fetch origin "$DEFAULT_BRANCH"
git checkout "$DEFAULT_BRANCH"

UNPUSHED=$(git rev-list "origin/${DEFAULT_BRANCH}..${DEFAULT_BRANCH}" --count)
if [ "$UNPUSHED" -gt 0 ]; then
    git rebase "origin/${DEFAULT_BRANCH}" || { git rebase --abort; exit 1; }
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

Branches outside `$BRANCH_FILTER` belong to another run — never touched.

## Step 4: Remove bounty worktrees (in scope)

```bash
for path in $(git worktree list --porcelain | awk '/^worktree / {print $2}'); do
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

Unmerged branches within the filter: `-D` only after user confirmation. Outside-filter branches: never touched.

## Step 6: Remove this run's state directory

Unless `--keep-state`:

```bash
rm -rf "$STATE_TARGET"

# --all with the whole .temp/bounty/ now empty
if [ -z "$RUN_ID" ] && [ -d .temp/bounty ] && [ -z "$(ls -A .temp/bounty 2>/dev/null)" ]; then
    rmdir .temp/bounty
fi

# Clear stale 'latest' pointer
if [ -n "$RUN_ID" ] && [ "$(readlink .temp/bounty/latest 2>/dev/null)" = "$RUN_ID" ]; then
    rm -f .temp/bounty/latest
fi

# Stray temp-file sweep (always bounty-owned)
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
echo "--- other bounty runs still present ---"
ls -la .temp/bounty 2>/dev/null | grep -vE '^(total|\.\.?$|latest)' || echo "(none)"
```

## Output

Report:

- Target run_id (or `--all`)
- PRs merged (numbers + titles)
- PR merge failures, if any, with the error
- Leak count per merged PR (files discarded)
- WIP branches created
- Worktrees removed
- Merged branches deleted
- Unmerged branches left in place
- `$STATE_TARGET` cleared or preserved (call out `plans/`, `fixes/`, `reviews/` explicitly)
- Other in-progress bounty runs still present
- Any stray temp files removed or left behind
