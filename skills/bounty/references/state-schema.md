# State Schema

All state lives under `.temp/bounty/`. The orchestrator reads and writes these files directly; agents write their outputs to them so every phase is resumable and auditable.

Directory layout:

```
.temp/bounty/
  config.json           # run config
  recon.md              # shared recon briefing (cached prompt context)
  claims/
    BUG-<lens>-<seq>.json
  votes/
    BUG-<id>.jsonl      # one JSON object per line, appended by voters
  leaderboard.json      # persistent scores
  queue.json            # fix queue
  fixes/
    BUG-<id>.json       # fix report
    BUG-<id>.patch      # unified diff
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
  "model_hunt": "sonnet",
  "model_fix": "opus",
  "started_at": "2026-04-19T14:22:00Z",
  "specialists": ["security", "concurrency", "performance", "nullsafety",
                  "errorhandling", "authz", "dataintegrity", "resources"]
}
```

## `claims/BUG-<id>.json`

```json
{
  "id": "BUG-security-001",
  "finder": "security",
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
  "co_discovered_by": []
}
```

Required fields: `id`, `finder`, `file`, `line`, `category`, `severity`, `evidence`, `trigger`, `impact`. Missing any → claim rejected by orchestrator before voting (no penalty, no credit).

`severity` ∈ {`low`, `medium`, `high`, `critical`}.

`fingerprint` is computed by the orchestrator: `sha256(normalize(file) + ":" + category + ":" + (line // 10))`. Collisions are merged — later submitters join `co_discovered_by`.

## `votes/BUG-<id>.jsonl`

One line per voter. Appended atomically. Example:

```json
{"voter":"concurrency","verdict":"VALID","reason":"Confirmed unescaped concat; trigger reproduces.","at":"2026-04-19T14:31:02Z"}
{"voter":"performance","verdict":"FALSE_POSITIVE","reason":"This method is unreachable from any route.","at":"2026-04-19T14:31:10Z"}
{"voter":"authz","verdict":"ABSTAIN","reason":"Need to verify the routing table; out of scope here.","at":"2026-04-19T14:31:15Z"}
```

Orchestrator tallies after all N−1 voters have written their lines.

## `leaderboard.json`

```json
{
  "updated_at": "2026-04-19T14:50:00Z",
  "scores": {
    "security":      {"found": 5, "confirmed": 4, "fp": 1, "fixes": 2, "score": 3},
    "concurrency":   {"found": 3, "confirmed": 3, "fp": 0, "fixes": 1, "score": 4},
    "performance":   {"found": 2, "confirmed": 2, "fp": 0, "fixes": 0, "score": 2},
    "nullsafety":    {"found": 0, "confirmed": 0, "fp": 0, "fixes": 0, "score": 0},
    "errorhandling": {"found": 0, "confirmed": 0, "fp": 0, "fixes": 0, "score": 0},
    "authz":         {"found": 0, "confirmed": 0, "fp": 0, "fixes": 0, "score": 0},
    "dataintegrity": {"found": 0, "confirmed": 0, "fp": 0, "fixes": 0, "score": 0},
    "resources":     {"found": 0, "confirmed": 0, "fp": 0, "fixes": 0, "score": 0}
  }
}
```

Score formula: `confirmed * 1 + fixes * 1 − fp * 3`.

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

## `fixes/BUG-<id>.json`

```json
{
  "bug_id": "BUG-security-001",
  "fixer": "security",
  "branch": "bounty/BUG-security-001-harden-ip-lookup",
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
  "reviews": [
    {"reviewer": "performance", "verdict": "APPROVE", "reason": "Minimal; parameterized."},
    {"reviewer": "nullsafety",  "verdict": "APPROVE", "reason": "No new null paths introduced."}
  ],
  "committed_at": "2026-04-19T15:02:00Z"
}
```

## Atomicity

Every file write uses a temp-file + rename pattern so readers never see a half-written file:

```bash
tmp=$(mktemp .temp/bounty/leaderboard.json.XXXX)
jq '.' > "$tmp" && mv "$tmp" .temp/bounty/leaderboard.json
```

This matters because multiple agents in worktrees may write to the same state directory concurrently.
