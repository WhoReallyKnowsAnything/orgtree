# Deferred Items — Phase 04 Plan 01

Out-of-scope discoveries found during execution, not fixed per the executor's
scope boundary (only auto-fix issues directly caused by the current task's changes).

## tests/update-rehearsal.test.mjs — pre-existing failures unrelated to this plan

Running the full `npm test` suite (which this plan's files are NOT part of)
surfaces pre-existing failures in `tests/update-rehearsal.test.mjs` /
`tools/rehearsal-isolation.mjs`: assertions constructing Windows-style test
fixture paths (`D:\...`, `E:\...`) expect an error message matching
`/inside the installed location/` but get "...inside the rehearsal output
directory..." instead. Neither this plan (04-01) nor any file it modifies
(packages/contracts/index.ts, apps/desktop/preload/index.ts,
apps/desktop/main/index.ts, apps/desktop/main/updater.ts,
apps/desktop/main/taskbar-attention.ts, apps/desktop/renderer/src/update-notice.tsx)
touches update-rehearsal.test.mjs or rehearsal-isolation.mjs. This predates
this plan's changes in this worktree and is out of scope — verification for
this plan instead ran the three specific test files this plan's tasks target
(tests/update-notice-ui.test.mjs, tests/updater.test.mjs,
tests/taskbarattention.test.mjs) plus `npm run typecheck`, both clean.
