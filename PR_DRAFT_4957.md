# PR draft: #4957

Branch: `arnoldokoye:fix/4957-windows-short-name-lock-dir` into `career-ops-hq:main`

## Before opening the upstream PR: get a Windows result

This change could not be run on Windows here. The upstream `Tests` workflow runs on `pull_request` to `main` with an ubuntu/macos/windows matrix, so to see the Windows leg first:

1. In the fork (arnoldokoye/career-ops), make sure Actions is enabled.
2. Open a PR inside the fork: `fix/4957-windows-short-name-lock-dir` into the fork's own `main`.
3. Check that the `test (windows-latest)` job passes, especially `go test ./...` with `TestLockDirectoryMatchesNodeForTheSameTracker` and `TestLockOverrideIsAcceptedAndRejectedIdentically`. Then close that fork PR.

If the Windows job fails, do not open the upstream PR. Bring the log back first.

## Comment to claim the issue (post on #4957 first)

> I'd like to take this, with the Node-side change you proposed: use `realpathSync.native` on Windows (which expands 8.3 short names, like Go's `EvalSymlinks`) for the tracker path and the temp root, and remove the Windows skip from the mirror test so the windows-latest CI leg checks it. I have a branch ready.

## PR title

fix(tracker-lock): expand Windows 8.3 short names so Go and Node share one lock

## PR body

## What does this PR do?

On Windows, `%TEMP%` is often the 8.3 short form (`C:\Users\RUNNER~1`). Go's `filepath.EvalSymlinks` expands it to the long form, while Node's `realpathSync` leaves it as written. The lock directory name is a hash of the canonical tracker path, so the Go dashboard and the Node writers (`set-status.mjs`, `merge-tracker.mjs`) hashed two spellings of the same tracker and did not exclude each other.

This PR adds `canonicalRealpath()` to `path-resolver.mjs`. It uses `realpathSync.native` on win32, which expands short names, and the JS `realpathSync` everywhere else, also as a fallback if the native call fails. It is used for:
- the tracker path (`canonicalizeTrackerPath`)
- the temp-dir root and the override's parent in `trackerLockDirFor` (`tracker-utils.mjs`)
- the web's synchronous mirror of the resolver (`web/src/lib/core/tracker-files.mjs`), so it stays in parity with the core

It also removes `skipOnWindowsPendingShortNameFix` from `dashboard/internal/data/tracker_lock_mirror_test.go`, so the existing Go/Node mirror test now runs on the `windows-latest` leg.

Effect on existing installs: on Windows, the lock directory for a short-name path changes once. Locks are transient, so a held lock is not carried across the upgrade, as the issue notes. POSIX output is unchanged, because the new branch only runs on win32.

Not covered: a tracker on a `subst` or mapped drive, where the native realpath resolves to the underlying volume. That case is not in the mirror corpus, and it was not covered before either.

## Related issue

Closes #4957

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
| `go test ./...` in `dashboard/` (Linux) | all packages ok, including the mirror tests |
| `node test-all.mjs --quick` (Node 24, Linux) | 15089 passed, 0 failed, 20 warnings (the same warnings as `main`) |
| `node --test web/tests/lib/tracker-files.test.mjs` | 1 pass, 0 fail |
| `npm run lint` | 1338 .mjs files passed syntax check |
| `test (windows-latest)` on the fork PR | **fill in before opening**: link to the run |

The Windows proof is the mirror test that was skipped on Windows before this change. On Linux the change is a no-op, so the Linux runs show nothing regressed; they cannot show the fix works.
