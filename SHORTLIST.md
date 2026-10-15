# Contribution shortlist (not for PR)

Checked against `career-ops-hq/career-ops` on 2026-10-15, upstream `main` at `49e14d4`.
For each issue: no assignee, no open PR found with `gh pr list --search`, and no claim in the comments.
Ranked by how ready it is to ship as a good PR for Arnold, then by fit (scoring, evals, grounding, parsing, tested refactors).

| Rank | Issue | Size | Status |
|---|---|---|---|
| 1 | #5300 Today lists a flagged posting as Recommended | S | Implemented on `fix/5300-today-flagged-posting` |
| 2 | No issue yet: web evaluate and pdf runs spawn core scripts from the data root | S/M | Open an issue first |
| 3 | #5125 Fact gate: unlisted German count noun is neither checked nor reported | M | Comment with the approach first |
| 4 | #5134 web: tie each evaluation's report to the run that wrote it | S | Probably fixed already; test-only PR or a close comment |
| 5 | #4798 atsLint/auditAts: partition ATS rules into structural and content-dependent | M/L | Design question first |

---

## 1. #5300 Today lists a flagged posting as Recommended

- Link: https://github.com/career-ops-hq/career-ops/issues/5300 (filed by the maintainer, santifer, today)
- What changes: Today's "Awaiting your decision" groups rows with `applyLineLabel(score)` and ignores legitimacy. A 4.0+ posting marked "Proceed with Caution" or "Suspicious" lands under Recommended and can become the "Apply next" button. The report page already shows "Check the posting first" for the same row. The fix reads the tier from each awaiting row's report header (the page already reads that report for the card reason), passes it to `groupAwaiting`, and adds a "Check the posting first" group. Without that group, a flagged row would drop off the page.
- Files: `web/src/lib/home/awaiting.mjs`, `web/src/app/page.tsx`, `web/src/components/home/today-dashboard.tsx`, `web/tests/lib/awaiting-order.test.mjs`
- Test plan: two new cases in `awaiting-order.test.mjs` (flagged 4.5 and 4.8 rows go to "Check the posting first"; rows with no tier group by score as before). The flagged case fails on `main`. Full suite, `tsc --noEmit` in `web/`, and a rendered check of `/` against a fixture data root.
- Size: S (4 files, +50/-7)
- Why it fits: scoring and ranking logic, maintainer-written with a named test file, so review should be quick.

## 2. No issue yet: web evaluate and pdf runs spawn core scripts from the data root

- Found while sizing #5134. Nobody has reported it.
- What is wrong: `persistEvaluation()` in `web/src/lib/report-persist.mjs` spawns `path.join(root, "reserve-report-num.mjs")` and the other persist scripts, and `/api/run` passes `root: careerOpsRoot()`, which is the **data** root. `renderAndMarkPdf()` in `web/src/lib/pdf-render.mjs` does the same for `generate-pdf.mjs` and `mark-pdf-ready.mjs`. With a split layout (`CAREER_OPS_ROOT` or `CAREER_OPS_DATA_DIR` pointing at a data-only directory), those scripts are not there, so a web evaluation fails at "Could not reserve a report number" and a web PDF run fails to render. Every other route resolves scripts with `rootScript()` (the code root). The evaluate half arrived today in `a98717e` (#3629); the pdf half is older.
- Proposed fix: add a `codeRoot` argument to `persistEvaluation` and `renderAndMarkPdf` for the script path, keep `cwd` as the data root (`persist-evaluation-report.mjs` writes under its cwd, and the other scripts resolve the data root from the inherited env), and pass `resolveCodeRoot(process.cwd(), process.env)` from `/api/run`.
- Files: `web/src/lib/report-persist.mjs`, `web/src/lib/pdf-render.mjs`, `web/src/app/api/run/route.ts`, `web/tests/lib/report-persist.test.mjs`, `web/tests/lib/pdf-render.test.mjs`
- Test plan: assert the spawned script path comes from `codeRoot` while `opts.cwd` stays the data root, in both modules.
- Size: S/M. It touches two modules and an area with an open user report (#4015 covers split-layout gaps in the web UI), so open an issue first and let the maintainer confirm the split layout is meant to work for runs. Then branch as `fix/<new-issue>-run-scripts-code-root`.
- Why it fits: an original finding with a clear repro is stronger for Arnold than picking up a pre-written issue.

## 3. #5125 Fact gate: a German count noun outside the list is neither checked nor reported

- Link: https://github.com/career-ops-hq/career-ops/issues/5125
- State: #5148 (merged) did the first half: it pinned the gap with a test and extended the German noun list. The issue stays open for the second half, and for French.
- What changes: `diagnoseCoverage()` in `verify-cv-facts.mjs` returns `null` as soon as `COUNT_CLAIM_RE` matches anything, so one listed noun silences the warning for every unlisted count in the document. Proposed: when at least one count matched and the document does not look English (`looksEnglish()` already exists), report the count-shaped spans that no `COUNT_CLAIM_RE` match covers. Flip the #5148 pin test to expect the warning, and add a French case.
- Files: `verify-cv-facts.mjs`, `tests/fact-gate-language-coverage.test.mjs`
- Test plan: German listed+unlisted on one line now warns on the unlisted span; the same for French; an English CV with an unlisted noun ("3 squads") keeps today's verdict; both example CVs still pass; `verify-cv-facts.mjs --self-test`.
- Size: M. Bigger than it looked: reporting every uncovered span would also start warning on English CVs with nouns outside the list, which is a policy change for every generated document. The language gate avoids that, but the maintainer should pick between "gate on language" and "extend the word list only" before code. The function's doc comment says it is deliberately conservative.
- Why it fits: grounding and fact checks, the core of the anti-fabrication gate.

## 4. #5134 web: tie each evaluation's report to the run that wrote it

- Link: https://github.com/career-ops-hq/career-ops/issues/5134
- State: probably fixed by `a98717e` (#3629, merged 2026-10-15). `/api/run` no longer diffs the reports directory. The run now gets its own number from `reserve-report-num.mjs` inside `persistEvaluation()` and sends `reportN: result.num`. `reportsBefore` and `hasNewCompletedReport` are gone from the route. The issue also asks for "a test with two overlapping runs where each one ends up with its own report number", and no such test exists.
- What changes: one test that runs two `persistEvaluation()` calls at once against the real `reserve-report-num.mjs` in a temp root and asserts two different numbers, each with its own report file.
- Files: a new `web/tests/lib/report-persist-overlap.test.mjs`
- Test plan: the test itself, then the full suite.
- Size: S, test-only. If the maintainer prefers, a short comment pointing at `a98717e` is enough to close the issue.
- Why it fits: data integrity and race testing.

## 5. #4798 atsLint/auditAts: partition the ATS rules into structural and content-dependent

- Link: https://github.com/career-ops-hq/career-ops/issues/4798
- What changes: `atsLint` in `cv-templates.mjs` and `auditAts` in `verify-ats.mjs` overlap. The issue asks to tag each rule as structural (valid on an unrendered template) or content-dependent (needs rendered output), so a template run reports content rules as skipped instead of passed. `templates/ats-rules.yml` already has a `detect: null` axis to copy.
- Files: `templates/ats-rules.yml`, `cv-templates.mjs`, `verify-ats.mjs`, their tests
- Test plan: every shipped template gets no critical "No email address found"; a content rule reports skipped on a template and runs on rendered HTML; the nested-table and hidden-text cases in the issue keep their current verdicts.
- Size: M/L, design-first. It decides which of two existing verdicts becomes project policy. A comment proposing the rule table is the right first step.
- Why it fits: ATS scoring and evals.

---

## Checked and left out

| Issue | Reason |
|---|---|
| #4963 h1b-sponsor displayClean() into lib/ | Already done by #5014 (merged); `plugins/h1b-sponsor/lib/clean.mjs` exists on main. The issue is still open, so a short comment pointing at #5014 would let a maintainer close it. |
| #4962 share parseScore | Claimed in a comment by Dakshx07 on 2026-10-11. |
| #5291 jd-skill-gap numbered items | Open PR #5293. |
| #4842 keyword-match list shapes | Open PR #4843. |
| #5179 check-jd-archive paraphrase | Open PR #5180. |
| #4980, #4978, #4982 provider tests | Open PRs #4981, #4979, #4984. |
| #5207 test-all data-root guard | Open PR #5279. |
| #5126 web/doctor personalization | Open PR #5311. |
| #5268 clean-markers joiners | Proposal; the author offers to send the PR once the shape is agreed. |
| #3103 title_filter word boundaries | Needs a maintainer decision, and the author has a PR attached. |
| #4890 AGENTS.md pointer to CONTRIBUTING.md | One-line doc change, but the author offered to send it. |
| #5000 check-liveness defects | All three are already fixed on main (Workable rung, launch failure, closure sentence). |
| #4221, #3967 | Fixed (#4323, #3959); a maintainer already said so in the comments. |
| #4795 extra Cursor skill directory | `.cursor/` no longer exists on main. |
| #5124, #5144, #5026 | Large, with design discussion in progress. |
| i18n issues | Skipped as asked. |

## Local test note

`node test-all.mjs` on Node 22.16 fails two web loader suites (`web-ts-alias-loader.test.mjs`) because Node 22.16 does not strip TypeScript types by default. CI runs Node 24. On Node 24.22, `node test-all.mjs --quick` on `main` passes (15096 passed, 0 failed). Go is not installed on this machine, so the dashboard build step is skipped with a warning. Before pushing, run the suite with Node 24 first on `PATH`, so the child suites it spawns use Node 24 too (for example `nvm install 24 && nvm use 24`).
