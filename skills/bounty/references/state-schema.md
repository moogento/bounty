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
  README.md             # live human-readable dashboard
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

## Atomicity

Every file write uses a temp-file + rename pattern so readers never see a half-written file:

```bash
tmp=$(mktemp .temp/bounty/leaderboard.json.XXXX)
jq '.' > "$tmp" && mv "$tmp" .temp/bounty/leaderboard.json
```

This matters because multiple agents in worktrees may write to the same state directory concurrently.
