# PR draft for #5300

Branch: `arnoldokoye:fix/5300-today-flagged-posting` into `career-ops-hq:main`
Commit: `fix(web): keep flagged postings out of Today's Recommended group`

## Issue comment to post first (claims the issue)

> I'd like to take this one. I'll pass each awaiting row's legitimacy (read from its report header) into `groupAwaiting`, add a "Check the posting first" group so flagged rows don't drop off Today, and add the flagged 4.5 case to `awaiting-order.test.mjs`.

## PR title

fix(web): keep flagged postings out of Today's Recommended group

## PR body

## What does this PR do?

Today's "Awaiting your decision" grouped rows by score alone, so a 4.0+ posting whose legitimacy is "Proceed with Caution" or "Suspicious" was listed under Recommended and could become the "Apply next" button. The report page shows "Check the posting first" for the same row. Today now passes each row's legitimacy to `applyLineLabel`, so both pages agree.

- `web/src/app/page.tsx` already reads each awaiting row's report for the card reason. It now also reads the `**Legitimacy:**` header with `parseReport()` and passes a `{ [n]: tier }` map to `TodayDashboard`.
- `groupAwaiting()` in `web/src/lib/home/awaiting.mjs` takes an optional `legitimacyOf(row)` lookup and hands the tier to `applyLineLabel`.
- `AWAITING_GROUPS` gains "Check the posting first", between Recommended and Below the apply line. Without it, a flagged row's label would match no group and the row would disappear from Today.
- A row with no tier (an older report without a Legitimacy line, or no report) groups by score as before.

## Related issue

Closes #5300

## Type of change

- [x] Bug fix
- [ ] New feature
- [ ] Documentation / translation
- [ ] Refactor (no behavior change)

## Test evidence

| Command | Result |
|---|---|
| `cd web && node --test tests/lib/awaiting-order.test.mjs` | 14 passed, 0 failed |
| Same suite against `main`'s `awaiting.mjs` | 13 passed, 1 failed: the new flagged-row case fails, as expected |
| `cd web && npx tsc --noEmit` | exit 0 |
| `npm run lint` | 1338 .mjs files passed syntax check |
| `node test-all.mjs` (Node 24.22) | 15096 passed, 0 failed, 18 warnings (one is "Dashboard build skipped — go compiler not in env") |

Manual check: `next dev` against a fixture data root with three Evaluated rows (4.6 "Proceed with Caution", 4.2 "High Confidence", 3.2 "High Confidence"). The rendered `/` shows Recommended: Clean Corp, Check the posting first: Flagged Corp, Below the apply line: Low Corp, and "Apply next" points at Clean Corp 4.2/5. With `main`'s score-only grouping, the 4.6 flagged row sorts first under Recommended, so it would be the "Apply next" target.

## Checklist

- [x] I have read [CONTRIBUTING.md](https://github.com/career-ops-hq/career-ops/blob/main/CONTRIBUTING.md)
- [x] If this is a new feature or architecture change, I opened an issue first (bug fixes, providers, docs & translations are exempt — send those straight in)
- [x] My PR does not include personal data (CV, email, real names, scan results, or pipeline data)
- [x] I ran `node test-all.mjs` and all tests pass
- [x] My changes respect the [Data Contract](https://github.com/career-ops-hq/career-ops/blob/main/DATA_CONTRACT.md) (no modifications to user-layer files)
- [x] My changes align with the [project roadmap](https://github.com/career-ops-hq/career-ops/discussions/156)

---
Questions? [Join the Discord](https://discord.gg/8pRpHETxa4) for faster feedback.
