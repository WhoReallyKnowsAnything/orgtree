# Deferred Items — Phase 02 (process-lifecycle-port)

## Plan 02-01

### Pre-existing `unittest discover` cross-contamination (out of scope)

`python -m unittest discover -s tests -p 'test_*.py'` (the plan's third
verification line) reports 91 failures / 478 errors / 25 skipped when run
across the whole `tests/` directory in a single interpreter process. This
is **not** caused by this plan's changes to `codexrun.py` / `antigravityrun.py`:

- The failing files (e.g. `test_work_payload_projections.py`,
  `test_worktree_helper.py`) do not import `codexrun` or `antigravityrun`
  at all.
- Several test modules (`test_codex_home_usage.py`,
  `test_provider_attempt_and_liveness.py`, and others) bind
  `orgtree.store.DATA_ROOT` from `ORGTREE_DATA` **at import time** — a
  pattern the codebase itself documents as fragile ("⚠ store.DATA_ROOT
  BINDS AT IMPORT TIME"). Running unrelated test files together in one
  `unittest discover` process races that binding: whichever module's
  `ORGTREE_DATA` value wins import order becomes the live root for every
  later-imported module, producing `DataRootDesync` errors and similar
  cross-test contamination unrelated to any single file's correctness.
- `test_worktree_helper.py`'s failure asserts a Windows path suffix
  (`endswith("new\\project")`) — an environment-specific (Windows-path-format)
  assertion failing on macOS, unrelated to process-tree termination.

**Verified in isolation:** both regression-guard files the plan calls out
by name pass cleanly when run as their own process, which is how the
plan's first two verification lines already run them:

    python -m unittest tests.test_codex_home_usage -v         # OK, 3 tests
    python -m unittest tests.test_provider_attempt_and_liveness -v   # OK, 23 tests
    python -m unittest tests.test_codexrun_process_tree tests.test_antigravityrun_process_tree -v  # OK, 4 tests

Per the executor's scope boundary, this pre-existing test-isolation
limitation is logged here and left unfixed — it predates this plan and
touches files outside `files_modified`.

## Plan 02-02

## 02-02: tests/test_mailhub_runtime.py — 6 pre-existing failures, out of scope

**Found during:** Task 2 regression check (`python -m unittest tests.test_mailhub_runtime -v`)

**Root cause:** `engine/mailhub` is a git submodule that is not checked out in
this worktree (`git submodule status` reports `-6e856eec8ccfbcf8d16451123903b9d2ce16450b
engine/mailhub`, and `ls engine/mailhub` is empty). Every test that spawns the
real `python -m mailhub.serve` child process fails with "the hub process
exited at startup (code 1)" because `mailhub.serve` cannot be imported.

**Confirmed unrelated to this plan's changes:** re-ran the same suite against
the unmodified (pre-fix) `engine/mailhub_runtime.py` via `git checkout HEAD --
engine/mailhub_runtime.py` — identical 8 passed / 6 FAILED result. The failure
predates and is independent of the `start_new_session`/`os.killpg` fix.

**Not fixed here:** initializing a git submodule is an environment/worktree
setup action outside this plan's scope (PROC-03 process-tree containment),
not a code defect in the files this plan touches.

**Failing tests (all pre-existing, all require the real submodule child):**
- a stale orphan pid file does not block a start
- a config change restarts the hub and persists
- an invalid patch is refused and the running hub is untouched
- (3 more with the same root cause — see full log)

## 02-02: `python -m unittest discover -s tests -p 'test_*.py'` — pre-existing, repo-wide, out of scope

**Found during:** Task 2 full-suite verification (plan's `<verification>` step 3)

**Symptom:** `unittest discover` across the whole `tests/` tree fails with
91 failures + 478 errors (1555 tests). Two structural causes visible in the
output: (1) several `orgtree.store`-backed test modules assert
`store.DATA_ROOT` starts with their own fixture's temp dir, but `store` binds
`DATA_ROOT` once at first import — whichever test module's `ORGTREE_DATA`
happened to be set when `store` first loads wins for the rest of the process,
so later modules see a mismatched `DATA_ROOT`; (2) some assertions are
Windows-path-shaped (e.g. `endswith("new\\project")`) and fail on this
macOS run regardless.

**Confirmed unrelated to this plan's changes:** temporarily moved
`tests/test_supervisor_kill_tree.py` and `tests/test_mailhub_orphan_reclaim.py`
out of `tests/` and re-ran discovery — same 91 failures / 478 errors / 25
skipped. This plan's two new test files are not a contributing cause.

**Not fixed here:** a cross-module global-state ordering issue and
Windows-only assertions spanning the entire existing suite are outside this
plan's scope (PROC-03 process-tree containment in 2 files). The individually
targeted verify commands for this plan's own tests
(`python -m unittest tests.test_supervisor_kill_tree -v` and
`python -m unittest tests.test_mailhub_orphan_reclaim -v`) both pass, as does
`tests.test_provider_attempt_and_liveness.SendPhaseOverRealPipes`, the
suite named in the plan's `<verification>` as the regression check most
relevant to supervisor.py's spawn/kill machinery.
