# Scoring Rules & Templates

## Scoring

| Event | Points |
|-------|--------|
| Claim confirmed | +1 |
| PoC reproduced (Phase 3.5) | +1 bonus |
| PoC failed to reproduce | −3 (demoted to FALSE_POSITIVE; flows through `fp`) |
| False positive (majority + FP quorum) | −3 |
| Fix approved (2/2 reviewers) | +1 |
| Fix secrets-scan hit (Step 5b.5) | −1 reversal on the implementer's fix credit |
| Fix rejected twice (abandoned) | 0 (no penalty, just lost the chance) |
| Co-discovery (same fingerprint as earlier claim) | 0 |

The −3 penalty is intentionally harsh: it should hurt more to submit junk than it rewards to confirm. This pushes agents toward quality over quantity.

## Thresholds

- **FP quorum**: `⌈(N−1)/3⌉` — at least a third of voters must call it out for the penalty to trigger. Combined with "majority is FP" so a single loud objector cannot tank a good claim.
- **Confirmation**: strict majority of non-abstaining voters say VALID. Ties go to INCONCLUSIVE (no points).
- **Fix approval**: both reviewers must approve. One reject sends it back.

## Leaderboard format

Render to stdout and to `.temp/bounty/README.md` after every resolution. The stdout render is the user-visible scoreboard — always emit it as assistant text, not a `Write` tool preview of `leaderboard.json`. The JSON file is written separately via Bash + minified `jq -cn` so its tool-call UI doesn't bury the table.

```
🏆 BOUNTY LEADERBOARD
┌─────┬──────────────────┬───────┬──────┬─────┬─────┬───────┬───────┐
│ #   │ Specialist       │ Found │ Conf │ PoC │ FPs │ Fixes │ SCORE │
├─────┼──────────────────┼───────┼──────┼─────┼─────┼───────┼───────┤
│ 🥇  │ 🧵 Concurrency   │     3 │    3 │   1 │   0 │     2 │    +6 │
│ 🥈  │ 🛡️  Security     │     5 │    4 │   2 │   1 │     1 │    +4 │
│ 🥉  │ ⚡ Performance   │     2 │    2 │   0 │   0 │     0 │    +2 │
│  4  │ ∅ NullSafety     │     2 │    1 │   0 │   0 │     1 │    +2 │
│  5  │ 🔐 AuthZ         │     3 │    1 │   0 │   1 │     0 │    -2 │
│  …  │ …                │     … │    … │   … │   … │     … │     … │
└─────┴──────────────────┴───────┴──────┴─────┴─────┴───────┴───────┘
 Confirmed: 11 │ PoC reproduced: 3 │ FPs: 2 │ Inconclusive: 4 │ Fixed: 4 │ Pending: 3
```

## PR body template

Use this when opening the aggregate PR via `gh pr create`. Keep the title format `<modulename>: <benefit>` per repo git-standards (no `fix:` / `feat:` prefixes).

### Title style — positive, module-generic, no bounty branding

The PR and merge titles will live in the long-term git log. They should read like normal product work, not as the output of a specific tool. **Do not mention "bounty", "bounty sweep", "bounty run", or numeric bug counts in the title.** Past title seen in the wild: `smartcart: Resolved 11 risk and reliability issues from bounty sweep` — that's bounty-branded and count-specific. Better: `smartcart: Reduced risk and improved reliability`.

Write titles as a benefit statement in the imperative-past ("Hardened X", "Reduced Y", "Improved Z", "Tightened …"), describing the *outcome* a reader cares about — not the bounty machinery that produced it. A reader six months from now should be able to read the title and care about what changed.

- Good: `blackpearl: Hardened order placement against replayed payment tokens`
- Good: `ledger: Tightened concurrency around balance updates`
- Good: `smartcart: Reduced risk and improved reliability`
- Bad:  `smartcart: Resolved 11 issues from bounty sweep`
- Bad:  `multiple: discovered and resolved N bugs via bounty run`
- Bad:  `bounty: fixed 7 security issues`

The "bounty run" framing belongs in the PR **body** (the "Bounty run summary" section below), not the title or any merge subject. The same rule applies to `gh pr merge --subject …` — if you set a custom squash subject, mirror the PR title style; do not reintroduce "bounty sweep".

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
- **Never mention "bounty", "bounty sweep", "bounty run", or bug counts in the title.** A title like `smartcart: Resolved 11 issues from bounty sweep` is count-specific and tool-branded; use `smartcart: Reduced risk and improved reliability` instead. The bounty framing lives in the commit body (BUG-ID reference) and the PR body, not the title.
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
