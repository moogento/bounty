# Scoring Rules & Templates

## Scoring

| Event | Points |
|-------|--------|
| Claim confirmed | +1 |
| False positive (majority + FP quorum) | −3 |
| Fix approved (2/2 reviewers) | +1 |
| Fix rejected twice (abandoned) | 0 (no penalty, just lost the chance) |
| Co-discovery (same fingerprint as earlier claim) | 0 |

The −3 penalty is intentionally harsh: it should hurt more to submit junk than it rewards to confirm. This pushes agents toward quality over quantity.

## Thresholds

- **FP quorum**: `⌈(N−1)/3⌉` — at least a third of voters must call it out for the penalty to trigger. Combined with "majority is FP" so a single loud objector cannot tank a good claim.
- **Confirmation**: strict majority of non-abstaining voters say VALID. Ties go to INCONCLUSIVE (no points).
- **Fix approval**: both reviewers must approve. One reject sends it back.

## Leaderboard format

Render to stdout and to `.temp/bounty/README.md` after every resolution:

```
🏆 BOUNTY LEADERBOARD
┌─────┬──────────────────┬───────┬──────┬─────┬───────┬───────┐
│ #   │ Specialist       │ Found │ Conf │ FPs │ Fixes │ SCORE │
├─────┼──────────────────┼───────┼──────┼─────┼───────┼───────┤
│ 🥇  │ 🧵 Concurrency   │     3 │    3 │   0 │     2 │    +5 │
│ 🥈  │ 🛡️  Security     │     5 │    4 │   1 │     1 │    +2 │
│ 🥉  │ ⚡ Performance   │     2 │    2 │   0 │     0 │    +2 │
│  4  │ ∅ NullSafety     │     2 │    1 │   0 │     1 │    +2 │
│  5  │ 🔐 AuthZ         │     3 │    1 │   1 │     0 │    -2 │
│  …  │ …                │     … │    … │   … │     … │     … │
└─────┴──────────────────┴───────┴──────┴─────┴───────┴───────┘
 Confirmed: 11 │ FPs: 2 │ Inconclusive: 4 │ Fixed: 4 │ Pending: 3
```

## PR body template

Use this when opening the aggregate PR via `gh pr create`. Keep the title format `<modulename>: <benefit>` per repo git-standards (no `fix:` / `feat:` prefixes).

```
## Bounty run summary

N specialists hunted **{scope}** and confirmed **{confirmed}** issues. **{fixed}** are resolved in this PR. **{critical}** critical items opened as separate issues for human review.

### Resolved

| BUG | Severity | Module | Specialist | Fixer |
|-----|----------|--------|------------|-------|
| BUG-001 | 🟠 high | blackpearl | 🛡️ Security | 🛡️ Security |
| BUG-002 | 🟡 medium | ledger | 🧵 Concurrency | ⚡ Performance |
| ... | | | | |

### Abandoned

- BUG-007 — fix rejected twice by reviewers; left unresolved. Reason: "invasive change touching 14 files; needs human design call."

### Critical issues opened

- #{issue-number} — {title}

### Validation

- phpcs: clean
- phpunit on affected modules: {pass_count}/{total} pass
- Manual review flagged: {high_severity_count} high-severity items — see inline commit messages
```

## Commit message rules

- Format: `<module>: <benefit-focused summary>` — lowercase module, no AI attribution
- Never use `fix`, `fixes`, `bug`, `error` in the title (per repo `.guidelines/git-standards.md`)
- Body: what the bug was in plain language, how the fix addresses it, any risk notes
- Reference the BUG-ID in the body, not the title

Example:

```
blackpearl: Hardened order-placement against replay of stored payment tokens

Payment tokens stored in the session were accepted for reuse across
orders, allowing an attacker with a leaked session to place arbitrary
orders against a previously-authorized card.

Tokens are now single-use and invalidated at the point of order
creation inside the existing transaction.

BUG-003 (bounty 2026-04-19)
```
