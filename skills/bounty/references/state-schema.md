# State Schema

All state lives under `.temp/bounty/`. The orchestrator reads and writes these files directly; agents write their outputs to them so every phase is resumable and auditable.

Directory layout:

```
.temp/bounty/
  config.json           # run config
  recon.md              # shared recon briefing (cached prompt context)
  claims/
    BUG-<lens>-<seed>-<seq>.json   # <seed> is the hunter prompt-seed index (1 when --hunters-per-lens=1)
  votes/
    BUG-<id>.jsonl      # one JSON object per line, appended by voters
  leaderboard.json      # persistent scores
  queue.json            # fix queue
  bundles.json          # bundle → bug-ids map (Phase 4)
  plans/
    <bundle>.md         # planner output per bundle, with YAML frontmatter
  fixes/
    BUG-<id>.json       # per-bug fix report (one per bug inside a bundle)
  reviews/
    <bundle>.json       # per-bundle review verdicts (one APPROVE/REJECT per bug)
  pr-review/
    <pr-number>.json    # Step 5e — per-PR automated-review feedback rounds (processed ids, verdicts, fix commits, rejection replies)
  dismissed.yml         # Phase 0d snapshot of the per-repo dismissed list (copy of .bounty/dismissed.yml at run start)
  README.md             # live human-readable dashboard
```

The **per-repo** dismissed list lives outside `$STATE_DIR` so it survives across runs:

```
<repo-root>/
  .bounty/
    dismissed.yml       # hand-editable; loaded at Phase 0d; appended after voter-confirmed FPs (opt-in)
```

## `config.json`

```json
{
  "scope": "src/blackpearl",
  "agents": 8,
  "severity_floor": "medium",
  "max_claims": 40,
  "no_fix": false,
  "hunters_per_lens": 1,
  "max_bundles": 6,
  "max_fixes": 10,
  "voting_mode": "auto",
  "state_dir": "/Users/jim/repos/moo-m2/.temp/bounty",
  "recon_budget_minutes": 5,
  "model_hunt": "sonnet",
  "model_vote": "haiku",
  "model_vote_effective": "haiku",
  "model_plan": "opus",
  "model_fix": "sonnet",
  "user_fix_scope": "all",
  "started_at": "2026-04-19T14:22:00Z",
  "specialists": ["security", "concurrency", "performance", "nullsafety",
                  "errorhandling", "authz", "dataintegrity", "resources"]
}
```

## `claims/BUG-<id>.json`

```json
{
  "id": "BUG-security-1-001",
  "finder": "security/1",
  "file": "src/blackpearl/Model/OrderValidator.php",
  "line": 142,
  "category": "SQL Injection",
  "severity": "high",
  "evidence": "  $this->connection->query(\"SELECT * FROM moo_nmsp_log WHERE ip='\" . $ip . \"'\")",
  "context_before": "  public function findByIp(string $ip) {",
  "context_after": "    return $this->connection->fetchAll();\n  }",
  "trigger": "Pass $ip = \"' OR 1=1 --\" via X-Forwarded-For header",
  "impact": "Arbitrary SELECT over the log table; exposes user IPs and timing data",
  "fingerprint": "sha256:5f2e…a810",
  "submitted_at": "2026-04-19T14:28:13Z",
  "co_discovered_by": ["security/2"],
  "collision_count": 2
}
```

Required fields: `id`, `finder`, `file`, `line`, `category`, `severity`, `evidence`, `trigger`, `impact`. Missing any → claim rejected by orchestrator before voting (no penalty, no credit).

`finder` format is `<lens>/<seed>` when `--hunters-per-lens > 1`, or just `<lens>` when `K = 1` (backwards compatible). `collision_count` defaults to `1` and is incremented by the orchestrator each time another hunter submits the same fingerprint.

`severity` ∈ {`low`, `medium`, `high`, `critical`}.

`fingerprint` is computed by the orchestrator: `sha256(normalize(file) + ":" + category + ":" + (line // 10))`. Collisions are merged — later submitters join `co_discovered_by`.

## `votes/BUG-<id>.jsonl` (full mode)

Written in `voting_mode: full`. One line per voter, appended atomically. Example:

```json
{"voter":"concurrency","verdict":"VALID","reason":"Confirmed unescaped concat; trigger reproduces.","at":"2026-04-19T14:31:02Z"}
{"voter":"performance","verdict":"FALSE_POSITIVE","reason":"This method is unreachable from any route.","at":"2026-04-19T14:31:10Z"}
{"voter":"authz","verdict":"ABSTAIN","reason":"Need to verify the routing table; out of scope here.","at":"2026-04-19T14:31:15Z"}
```

Orchestrator tallies after all N−1 voters have written their lines.

## `votes/VOTER-<lens>.jsonl` (panel mode)

Written in `voting_mode: panel`. One line per `(voter, bug_id)` pair. Each line includes the `bug_id` so the orchestrator can group by claim when tallying.

```json
{"bug_id":"BUG-security-1-003","voter":"concurrency","verdict":"VALID","reason":"Confirmed unescaped concat.","at":"2026-04-19T14:31:02Z"}
{"bug_id":"BUG-security-1-004","voter":"concurrency","verdict":"FALSE_POSITIVE","reason":"Method is unreachable.","at":"2026-04-19T14:31:10Z"}
```

## `tally.json`

The orchestrator's aggregate of every vote (full or panel), keyed by `bug_id`. Refreshed after each voter finishes:

```json
{
  "updated_at": "2026-04-19T14:45:00Z",
  "tallies": {
    "BUG-security-1-003": {
      "valid": 5, "false_positive": 0, "abstain": 2,
      "resolution": "CONFIRMED",
      "voters": ["concurrency", "performance", "nullsafety", "errorhandling", "dataintegrity", "authz", "resources"]
    }
  }
}
```

## `leaderboard.json`

```json
{
  "updated_at": "2026-04-19T14:50:00Z",
  "scores": {
    "security":      {"found": 5, "confirmed": 4, "fp": 1, "collisions": 2, "fixes": 2, "score": 3},
    "concurrency":   {"found": 3, "confirmed": 3, "fp": 0, "collisions": 1, "fixes": 1, "score": 4},
    "performance":   {"found": 2, "confirmed": 2, "fp": 0, "collisions": 0, "fixes": 0, "score": 2},
    "nullsafety":    {"found": 0, "confirmed": 0, "fp": 0, "collisions": 0, "fixes": 0, "score": 0},
    "errorhandling": {"found": 0, "confirmed": 0, "fp": 0, "collisions": 0, "fixes": 0, "score": 0},
    "authz":         {"found": 0, "confirmed": 0, "fp": 0, "collisions": 0, "fixes": 0, "score": 0},
    "dataintegrity": {"found": 0, "confirmed": 0, "fp": 0, "collisions": 0, "fixes": 0, "score": 0},
    "resources":     {"found": 0, "confirmed": 0, "fp": 0, "collisions": 0, "fixes": 0, "score": 0}
  }
}
```

Score formula: `confirmed * 1 + fixes * 1 − fp * 3`. `collisions` is the number of claims on this lens where `collision_count ≥ 2` (pre-validated by an independent same-lens hunter); it is a confidence signal only, not a scoring input.

## `queue.json`

```json
{
  "pending":  ["BUG-security-001", "BUG-concurrency-002"],
  "in_progress": {
    "BUG-concurrency-001": {"assigned": "concurrency", "started_at": "2026-04-19T14:45:00Z"}
  },
  "done": ["BUG-performance-001"],
  "abandoned": {
    "BUG-authz-002": {"reason": "rejected twice — needs human design call"}
  }
}
```

## `bundles.json`

Written at the start of Phase 4. Maps each bundle slug to the bug IDs it contains and the *guessed* test strategy; the authoritative `test_strategy` is written later by the planner into the plan frontmatter.

```json
{
  "updated_at": "2026-04-19T15:00:00Z",
  "bundles": {
    "fix-acl": {
      "bug_ids": ["BUG-authz-1-001", "BUG-authz-1-002", "BUG-authz-1-004"],
      "guessed_strategy": "tdd",
      "planner_assigned": "authz",
      "implementer_assigned": "authz",
      "branch": "bounty/fix-acl"
    },
    "fix-concurrency": {
      "bug_ids": ["BUG-concurrency-1-001", "BUG-concurrency-1-002"],
      "guessed_strategy": "architectural",
      "planner_assigned": "concurrency",
      "implementer_assigned": "concurrency",
      "branch": "bounty/fix-concurrency"
    }
  }
}
```

## `plans/<bundle>.md`

Markdown file with YAML frontmatter. The planner is read-only — if a planner's worktree has any file changes on completion, the plan is rejected and the bundle falls back to per-bug fixing.

```markdown
---
bundle: fix-acl
test_strategy: tdd          # tdd | architectural — picks reviewer model
commit_order:
  - BUG-authz-1-001
  - BUG-authz-1-002
  - BUG-authz-1-004
planner: authz
written_at: 2026-04-19T15:10:00Z
---

# fix-acl plan

## BUG-authz-1-001 — admin endpoint missing ACL
- Proposed fix: add `<resource id="…" />` to `etc/acl.xml`; add `_isAllowed()` override in controller
- Test stance: feature test — unauthenticated request should 403 after change, was 200 before
- Dependencies: none

## BUG-authz-1-002 — …
```

## `reviews/<bundle>.json`

Written by the bundle reviewer after Step 4d. One verdict per bug in the bundle.

```json
{
  "bundle": "fix-acl",
  "reviewer": "errorhandling",
  "reviewer_model": "haiku",
  "reviewed_at": "2026-04-19T15:30:00Z",
  "verdicts": [
    {"bug_id": "BUG-authz-1-001", "verdict": "APPROVE", "reason": "ACL entry + controller guard; test covers the path."},
    {"bug_id": "BUG-authz-1-002", "verdict": "APPROVE", "reason": "Minimal; matches repo conventions."},
    {"bug_id": "BUG-authz-1-004", "verdict": "REJECT",  "reason": "Guard added but mass-assignment still possible via `setData`."}
  ]
}
```

## `pr-review/<pr-number>.json`

Written by the Step 5e PR-feedback worker. One file per PR, one `rounds[]` entry per round. `exit_reason` is set once the worker stops looping.

```json
{
  "pr_number": 1234,
  "branch": "bounty/20260420-143000-smartcart/pr-1-acl-and-csrf",
  "started_at": "2026-04-20T04:05:00Z",
  "exit_reason": "no_new_feedback",
  "rounds": [
    {
      "round": 1,
      "wait_seconds": 300,
      "checks_at_start": {"passed": 3, "failed": 0, "pending": 2},
      "processed_comment_ids": [892341, 892352, 892399],
      "evaluated": [
        {"comment_id": 892341, "user": "coderabbitai[bot]", "path": "app/Http/Orders.php", "line": 142, "verdict": "VALID",   "reason": "Genuine SQL concat bypass; switched to prepared statement.", "applied": true},
        {"comment_id": 892352, "user": "coderabbitai[bot]", "path": "app/Http/Cart.php",   "line":  63, "verdict": "INVALID", "reason": "Conflicts with ADR-017 live-pricing contract.",          "applied": false},
        {"comment_id": 892399, "user": "github-actions[bot]", "path": null, "line": null, "verdict": "SKIP",    "reason": "Informational summary only.",                          "applied": false}
      ],
      "fix_commit_shas": ["a1b2c3d"],
      "rejection_reply_comment_id": 892410,
      "pushed": true,
      "finished_at": "2026-04-20T04:12:00Z"
    }
  ]
}
```

`exit_reason` ∈ {`no_new_feedback`, `all_invalid_or_skip`, `max_rounds_reached`, `aborted_failed_validation`}.

`evaluated[].applied` captures whether a `VALID` suggestion was actually committed (a VALID fix whose validation regresses ends up `applied: false` with a rationale reply posted).

## `fixes/BUG-<id>.json`

```json
{
  "bug_id": "BUG-security-1-001",
  "bundle": "fix-xss-injection",
  "implementer": "security",
  "branch": "bounty/fix-xss-injection",
  "commit_sha": "a1b2c3d",
  "status": "committed",
  "files_changed": [
    "src/blackpearl/Model/OrderValidator.php"
  ],
  "test_added": true,
  "test_path": "src/blackpearl/Test/Unit/Model/OrderValidatorTest.php",
  "validation": {
    "phpcs": "clean",
    "phpunit": "12/12 pass",
    "manual_notes": "Trigger fails after fix — confirmed."
  },
  "committed_at": "2026-04-19T15:02:00Z"
}
```

`status` ∈ {`committed`, `skipped`, `reverted`}. `skipped` means the implementer couldn't make the bug pass and logged a reason. `reverted` is set by the orchestrator after a Step 4d rejection drops the commit from the bundle branch. Reviewer verdicts are held in `reviews/<bundle>.json`, not duplicated here.

## `.bounty/dismissed.yml` (per-repo, outside `$STATE_DIR`)

Hand-editable memory of patterns previously determined NOT to be bugs in this codebase. Lives under source control so it persists across bounty runs and team members. Phase 0d loads it and snapshots a copy into `$STATE_DIR/dismissed.yml` for the run.

```yaml
version: 1
dismissed:
  - id: MOO-001
    added: 2026-04-20
    run_id: 20260420-030000-smartcategories     # optional — the run that first surfaced it
    files:
      - "app/code/Moogento/**/view/adminhtml/templates/system/config/hint.phtml"
    categories: ["security", "xss", "supply-chain"]
    title: Raw HTML in Moogento hint templates
    rationale: |
      Moogento controls the upstream content shown in these hint.phtml
      templates; raw HTML is intentional for formatting (links, emphasis,
      occasional images). The Moo.php null guard is the relevant safeguard
      around missing keys — escaping the HTML would break rendering with no
      security benefit.
```

Field semantics:

- `id` — stable identifier. Orchestrator auto-assigns `<SCOPE-PREFIX>-NNN` when appending; humans can edit.
- `files` — fnmatch-style globs, matched against `claim.file` after normalization.
- `categories` — list of category tokens. Match is substring, case-insensitive, any-overlap.
- `title` — one-line human label (used in the auto-FP log line).
- `rationale` — free-text prose. Included verbatim in hunter prompts so the hunter can judge whether a specific claim overrides the dismissal.
- `added`, `run_id` — provenance. `added` is required on new entries; `run_id` is optional.

Matching rule for "should this claim be auto-FP'd?":

```
match = file_matches_any(claim.file, entry.files)
    AND categories_overlap(claim.category, entry.categories)
```

Both must be true. A partial match (one side only) surfaces as a debug log line but does not auto-dismiss — the claim still goes to voting.

## Atomicity

Every file write uses a temp-file + rename pattern so readers never see a half-written file:

```bash
tmp=$(mktemp .temp/bounty/leaderboard.json.XXXX)
jq '.' > "$tmp" && mv "$tmp" .temp/bounty/leaderboard.json
```

This matters because multiple agents in worktrees may write to the same state directory concurrently. The per-repo `.bounty/dismissed.yml` uses the same pattern when the orchestrator appends new entries.
