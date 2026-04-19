---
description: Run a competitive multi-agent bug bounty across the codebase
argument-hint: "[--scope <path>] [--agents N] [--severity low|medium|high|critical] [--max-claims N] [--no-fix]"
allowed-tools: Task, Bash, Read, Write, Edit, Glob, Grep, Skill
---

# Bounty

Launch the `bounty` skill with `$ARGUMENTS`.

Invoke the skill and follow its phased workflow. Do not summarize or skip phases — the skill's structure is load-bearing (each phase writes state that later phases read).

Before starting, confirm with the user:
- The resolved scope (if not given, ask whether to hunt the whole repo or a module)
- The severity floor (default `medium`)
- Whether fixes should be committed (default yes; use `--no-fix` for report-only)

After the run, remind the user to use `/bounty-cleanup` to tear down worktrees and `bounty/*` branches that were merged.
