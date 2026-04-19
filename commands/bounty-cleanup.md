---
description: Clean up worktrees, branches, and state from a bounty run
argument-hint: "[--keep-state]"
allowed-tools: Bash(git:*), Bash(rm:*), Bash(ls:*), Read
model: haiku
---

# Bounty Cleanup

Tear down artifacts from a bounty run.

## Step 1: List bounty worktrees and branches

```bash
git worktree list
git branch --list 'bounty/*'
```

## Step 2: Identify merged branches

For each `bounty/*` branch, check if it is merged into the current `main`:

```bash
for b in $(git branch --list 'bounty/*' --format='%(refname:short)'); do
  if git merge-base --is-ancestor "$b" main 2>/dev/null; then
    echo "MERGED  $b"
  else
    echo "UNMERGED $b"
  fi
done
```

Show the list to the user and confirm before deleting **any unmerged branches**. Merged branches can be deleted without prompting.

## Step 3: Remove worktrees

Remove every worktree whose path is under `.git/.worktrees/` (or wherever the system placed bounty worktrees — check `git worktree list`):

```bash
for path in $(git worktree list --porcelain | awk '/^worktree / {print $2}' | grep -E 'bounty|worktree-agent'); do
  git worktree remove "$path" --force
done
git worktree prune
```

## Step 4: Delete branches

```bash
# Merged branches — safe delete
for b in $(git branch --list 'bounty/*' --format='%(refname:short)' --merged main); do
  git branch -d "$b"
done

# Unmerged branches — only after user confirmed
# for b in …; do git branch -D "$b"; done
```

## Step 5: Remove state directory

Unless `--keep-state` was given:

```bash
rm -rf .temp/bounty
```

If `--keep-state` was given, leave `.temp/bounty/` intact so the user can audit the run later.

## Step 6: Verify

```bash
git worktree list
git branch --list 'bounty/*' | wc -l
ls -la .temp/bounty 2>/dev/null || echo "state directory cleared"
```

## Output

Report:
- Worktrees removed
- Merged branches deleted
- Unmerged branches left (if any) with a reminder that they are the user's to decide on
- Whether state was cleared or preserved
