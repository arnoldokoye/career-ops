# PR draft: #5125

Branch: `arnoldokoye:fix/5125-fact-gate-unlisted-german-count` into `career-ops-hq:main`

## Comment to claim the issue (post on #5125 first)

> I'd like to take the remaining gap here. When a German document has at least one recognized count, the coverage warning would list the count nouns the extractor did not read, instead of staying silent. English documents are unaffected. I have a branch with tests ready and will open the PR shortly.

## PR title

fix(fact-gate): report an unlisted German count noun next to a listed one

## PR body

## What does this PR do?

In a German document, one listed count noun ("3 Standorten") silenced the coverage warning for the whole document, so an unlisted noun next to it ("45 Spezialisten") was neither checked against the sources nor reported. Now, when a recognized count noun is German, `diagnoseCoverage()` reports every count-shaped span the extractor did not read, under the new reason `count-nouns-not-recognized`.

Details:
- The German nouns move from `METRIC_NOUNS` into their own `GERMAN_METRIC_NOUNS` list, which `METRIC_NOUNS` spreads in. The regex and claim extraction are unchanged. `posts` stays in `METRIC_NOUNS` because it is English.
- `countShapedSpans()` now returns each span with the offset of its number. A span is "read" when a recognized `COUNT_CLAIM_RE` match starts at the same offset.
- English documents are unaffected, since they have no German noun to trigger this. A German document whose counts are all listed still reports nothing.
- The gate never blocks on this: like the existing coverage warning, it can only turn a `pass` into a `warn`.
- Out of scope: French has the same gap but no French noun list yet, so the documented "coincidental coverage" blind spot for French is unchanged.

## Related issue

Closes #5125

## Type of change

- [x] Bug fix
- [ ] New feature
- [ ] Documentation / translation
- [ ] Refactor (no behavior change)

## Checklist

- [x] I have read [CONTRIBUTING.md](https://github.com/career-ops-hq/career-ops/blob/main/CONTRIBUTING.md)
- [x] If this is a new feature or architecture change, I opened an issue first (bug fixes, providers, docs & translations are exempt — send those straight in)
- [x] My PR does not include personal data (CV, email, real names, scan results, or pipeline data)
- [x] I ran `node test-all.mjs` and all tests pass
- [x] My changes respect the [Data Contract](https://github.com/career-ops-hq/career-ops/blob/main/DATA_CONTRACT.md) (no modifications to user-layer files)
- [x] My changes align with the [project roadmap](https://github.com/career-ops-hq/career-ops/discussions/156)

## Test evidence

| Command | Result |
|---|---|
| `node --test tests/fact-gate-language-coverage.test.mjs` | 11 pass, 0 fail |
| `node verify-cv-facts.mjs --self-test` | 136 passed, 0 failed |
| `npm run lint` | 1338 .mjs files passed syntax check |
| `node test-all.mjs --quick` (Node 24) | 15089 passed, 0 failed, 20 warnings (the same warnings as `main`) |

Test changes in `tests/fact-gate-language-coverage.test.mjs`:
- The test that pinned the gap now asserts `coverage.reason === 'count-nouns-not-recognized'` with spans `['45 Spezialisten']`. With an English source that backs the listed count, the verdict is now `warn`, not `pass`.
- New test: `Built 12 dashboards for 40 users.`, `Wrote 12 posts and ran 40 workshops for 9 cohorts.` and `Leitete 45 Mitarbeiter an 3 Standorten.` all keep `coverage: null`.

On Node 22.22 one unrelated harness self-test (`tests/test-all-jobs.test.mjs:121`) fails on untouched `main` too. It passes on Node 24.
