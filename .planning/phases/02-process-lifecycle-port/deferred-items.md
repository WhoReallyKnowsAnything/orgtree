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
