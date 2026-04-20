---
description: Run a competitive multi-agent bug bounty across the codebase
argument-hint: "[--scope <path>] [--agents N] [--severity low|medium|high|critical] [--max-claims N] [--no-fix] [--fresh|--resume] [--max-bugs-per-pr N] [--max-lines-per-pr N]"
allowed-tools: Task, Bash, Read, Write, Edit, Glob, Grep, Skill
---

# Bounty

Launch the `bounty` skill with `$ARGUMENTS`.

Invoke the skill and follow its phased workflow. Do not summarize or skip phases — the skill's structure is load-bearing (each phase writes state that later phases read).

**The championship scoreboard (skill Phase 0e) is the first thing the user should see.** Previous runs have silently skipped it and jumped straight to "dispatching recon…". That is a skill violation — emit the empty-scoreboard table on stdout before any other output, then re-render it at every phase boundary and every scoring event.

Before starting, confirm with the user:
- The resolved scope (if not given, ask whether to hunt the whole repo or a module)
- The severity floor (default `medium`)
- Whether fixes should be committed (default yes; use `--no-fix` for report-only)

After the run, remind the user to use `/bounty-cleanup` to tear down worktrees and `bounty/*` branches that were merged.
