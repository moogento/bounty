# Agent Specialties

Eight specialist lenses. Each hunter gets one as a starting focus. They may still claim bugs outside their lens — duplicates across lenses are deduplicated by fingerprint, not by specialty.

Pick the first N specialties from this list when `--agents N` is set (N < 8), or cycle through when N > 8.

---

## 🛡️ Security

Subagent preference: `moo-security-reviewer` if available, else `general-purpose`.

Focus patterns:
- SQL injection — string-concatenated queries, unescaped params in `Zend_Db`, raw SQL in Magento collections
- XSS — unescaped output in `.phtml`, missing `$block->escapeHtml()`, `escapeJs()`, `escapeUrl()`
- Open redirects — unchecked `redirectUrl` params
- Path traversal — file operations on user-controlled paths without realpath checks
- Secrets in code — hardcoded API keys, tokens, bcrypt salts
- CSRF gaps — admin controllers missing form keys
- Unsafe deserialization — `unserialize()` on untrusted input

## 🧵 Concurrency & timing

Focus patterns:
- Race conditions on shared DB rows (order status, stock, payment) without `SELECT … FOR UPDATE` or Magento's `lockManager`
- TOCTOU — check-then-act patterns without locks
- Missing transaction boundaries around multi-write operations
- Queue consumers that can process the same message twice (no idempotency key)
- Cache stampede — expiry without single-flight
- Event observers that depend on observer ordering

## ⚡ Performance

Focus patterns:
- N+1 in collection iteration, especially `foreach ($collection as $item) { $item->load(…) }`
- Unbounded queries — `getCollection()` without `setPageSize`
- Missing DB indexes on filter/join columns
- O(n²) loops on request-hot paths
- Large payloads loaded into memory when streamable
- Unnecessary cache misses (cache key collisions, TTL=0)

## ∅ Null safety & type correctness

Focus patterns:
- Methods returning `?T` whose callers don't check for null
- Array accesses on possibly-empty results (`$arr[0]` after `array_filter`)
- Unvalidated external input flowing into strict-typed methods
- PHP 8 `readonly`/typed-property initialization order bugs
- Enum/ADT handling that misses a case

## ⚠️ Error handling

Focus patterns:
- `catch (\Exception $e) {}` — swallowed exceptions with no log, no rethrow
- Generic `catch` hiding transient vs. fatal errors
- Missing rollbacks after exceptions inside transactions
- Error return values ignored (`false`, `null`) with no branching
- Mis-scoped `try` — catching exceptions from code that wasn't meant to be caught
- HTTP responses that return 200 on failure

## 🔐 AuthN / AuthZ

Focus patterns:
- Admin-only endpoints missing ACL declaration in `acl.xml` or the controller
- Customer-owned resources accessed by ID without ownership check
- Role elevation via mass-assignment (setData of `is_admin`, `role_id`)
- Session fixation, missing session regeneration after login
- API tokens not rotated on password change
- GraphQL resolvers bypassing store/customer context

## 🗄️ Data integrity

Focus patterns:
- Missing foreign keys / cascade rules in `db_schema.xml`
- Columns that should be NOT NULL but aren't
- Migrations that drop columns without data preservation
- Double-writes that can diverge (DB + cache, DB + search index)
- Money/decimal stored as float
- Timezone-naive datetimes

## 📦 Resource management

Focus patterns:
- File handles / sockets / DB connections not closed on error paths
- Streams opened in loops without `fclose`
- `tempnam()` files never cleaned up
- Unbounded memory accumulators (arrays growing forever in long-running processes)
- Process forks without reaping
- Growable log files with no rotation
