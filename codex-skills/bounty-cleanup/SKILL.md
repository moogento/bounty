---
name: bounty-cleanup
description: Use when the user wants to remove bounty worktrees, `bounty/*` branches, or `.temp/bounty/` state after a run.
metadata:
  short-description: Clean up bounty branches, worktrees, and state
---

# Bounty Cleanup

Clean up artifacts from a bounty run without disturbing unrelated work.

## Safety Rules

1. Delete merged `bounty/*` branches without prompting.
2. Ask before deleting any unmerged `bounty/*` branch.
3. Preserve `.temp/bounty/` when the user says `--keep-state` or otherwise asks to keep the audit trail.
4. Do not touch non-bounty branches or unrelated worktrees.

## Step 1: Inspect current state

Collect:

```bash
git worktree list
git branch --list 'bounty/*'
ls -la .temp/bounty
```

If `.temp/bounty/` does not exist, continue. Cleanup may still be needed for branches or worktrees.

## Step 2: Classify branches

Determine which `bounty/*` branches are already merged into `main`:

```bash
for b in $(git branch --list 'bounty/*' --format='%(refname:short)'); do
  if git merge-base --is-ancestor "$b" main 2>/dev/null; then
    echo "MERGED  $b"
  else
    echo "UNMERGED $b"
  fi
done
```

Show the merged and unmerged lists clearly before deleting anything unmerged.

## Step 3: Remove bounty worktrees

Remove worktrees that obviously belong to bounty runs:

```bash
for path in $(git worktree list --porcelain | awk '/^worktree / {print $2}' | grep -E 'bounty|worktree-agent'); do
  git worktree remove "$path" --force
done
git worktree prune
```

If a worktree path does not clearly belong to bounty, leave it alone and mention it.

## Step 4: Delete merged branches

Delete merged bounty branches:

```bash
for b in $(git branch --list 'bounty/*' --format='%(refname:short)' --merged main); do
  git branch -d "$b"
done
```

For unmerged branches, ask first. Only use `git branch -D` after the user confirms.

## Step 5: Remove state directory

Unless the user asked to keep state:

```bash
rm -rf .temp/bounty
```

If state is preserved, say so explicitly.

## Step 6: Verify

Run:

```bash
git worktree list
git branch --list 'bounty/*' | wc -l
ls -la .temp/bounty 2>/dev/null || echo "state directory cleared"
```

## Output

Report:

- worktrees removed
- merged branches deleted
- unmerged branches left in place
- whether `.temp/bounty/` was cleared or preserved
