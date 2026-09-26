---
phase: 02-process-lifecycle-port
plan: 01
subsystem: process-lifecycle
tags: [subprocess, process-group, posix, sigkill, termination, codexrun, antigravityrun]
requires: []
provides: [codexrun-close-reaps-process-tree, antigravityrun-kill-tree-reaps-process-tree]
affects: [engine/backend/orgtree/codexrun.py, engine/backend/orgtree/antigravityrun.py]
tech-stack:
  added: []
  patterns: [start_new_session-posix-process-group, os.killpg-sigkill-termination]
key-files:
  created:
    - tests/test_codexrun_process_tree.py
    - tests/test_antigravityrun_process_tree.py
  modified:
    - engine/backend/orgtree/codexrun.py
    - engine/backend/orgtree/antigravityrun.py
decisions:
  - "Mirrored gitrunner.py's proven _stop() pattern exactly: start_new_session=(os.name != \"nt\") on spawn, os.killpg(pid, signal.SIGKILL) wrapped in a narrow except on the POSIX kill path, Windows taskkill branch untouched."
  - "Kept each file's existing unconditional fallback kill (codexrun's belt-kill, antigravity's proc.wait) untouched — the fix only adds the POSIX-branch os.killpg call, it does not restructure the surrounding kill/wait sequence."
  - "Each new test file includes a frozen 'control' test replicating the exact pre-fix Popen/kill shape independent of the (now-fixed) production code, so it keeps demonstrating the vulnerability class as permanent regression evidence even after the fix lands, alongside the RED-before/GREEN-after test that drives the real production close()/kill_tree() functions."
metrics:
  duration: "~35 min"
  completed: 2026-09-23
status: complete
actuals:
  tokens: 3354
  tasks: 2
  commits: 4
---

# Phase 02 Plan 01: Process-Tree Termination for codexrun.py and antigravityrun.py Summary

Ported `codexrun.py`'s `AppServerClient.close()` and `antigravityrun.py`'s `kill_tree()` from parent-only termination to POSIX process-group termination (`start_new_session` + `os.killpg(pid, SIGKILL)`), matching `gitrunner.py`'s already-proven pattern, so "stop" no longer orphans a spawned CLI's child processes.

## What Was Built

**Task 1 — codexrun.py:**
- Added `import signal` to the top-level imports.
- `AppServerClient.__init__`'s `Popen(...)` call now passes `start_new_session=(os.name != "nt")`.
- `close()`'s kill sequence now branches: `os.name == "nt"` keeps the existing `taskkill /T /F` call unchanged; the new `else` branch adds `os.killpg(self.proc.pid, signal.SIGKILL)` inside `try/except ProcessLookupError: pass`. The existing unconditional `self.proc.kill()` "belt" and `self.proc.wait(timeout=5)` are untouched.
- New `tests/test_codexrun_process_tree.py`: `test_close_reaps_grandchild_via_process_group` constructs a real `AppServerClient` (via the `argv_head` seam, spawning `/bin/sh -c "sleep 30 & echo $! > pidfile; wait"` in place of the real `codex` binary) and asserts the backgrounded `sleep` grandchild is dead within 2s of `close()`. `test_prefix_kill_pattern_orphans_grandchild` is a frozen control replicating the exact pre-fix Popen/kill shape, permanent regression evidence independent of production code state.

**Task 2 — antigravityrun.py:**
- Added `import signal` to the top-level imports.
- `AntigravityTurn.launch()`'s `Popen(...)` call now passes `start_new_session=(os.name != "nt")`.
- Module-level `kill_tree(proc)`'s kill sequence now branches: the `if os.name == "nt": taskkill ...` branch is unchanged; the new `else` branch replaces the old unconditional `proc.kill()` with `os.killpg(proc.pid, signal.SIGKILL)`, still inside the existing `try/except (OSError, subprocess.TimeoutExpired)` (which already covers `ProcessLookupError`, an `OSError` subclass). The trailing `proc.wait(timeout=15)` is untouched.
- New `tests/test_antigravityrun_process_tree.py`: `test_kill_tree_reaps_grandchild_via_process_group` spawns a real `/bin/sh -c "sleep 30 & echo $! > pidfile; wait"` child with `start_new_session=True`, calls `antigravityrun.kill_tree(proc)` directly, and asserts the grandchild is dead within 2s. `test_prefix_kill_pattern_orphans_grandchild` is the same frozen control pattern as Task 1.

## TDD Gate Compliance

Both tasks followed the RED → GREEN cycle:

| Task | RED commit | GREEN commit |
|------|-----------|---------------|
| 1 (codexrun.py) | `7c09aee` test(02-01): add failing process-tree test for codexrun.py close() | `d505ae8` feat(02-01): codexrun.py — start_new_session spawn + os.killpg termination |
| 2 (antigravityrun.py) | `d1d9552` test(02-01): add failing process-tree test for antigravityrun.py kill_tree() | `e3ca4ed` feat(02-01): antigravityrun.py — start_new_session spawn + os.killpg termination |

Both RED commits were verified failing (`AssertionError: ... left the grandchild running`) before their matching GREEN commits were verified passing.

## Verification

```
python -m unittest tests.test_codexrun_process_tree -v          # OK, 2 tests
python -m unittest tests.test_antigravityrun_process_tree -v    # OK, 2 tests
python -m unittest tests.test_codex_home_usage -v               # OK, 3 tests (plan-named regression guard)
python -m unittest tests.test_provider_attempt_and_liveness -v  # OK, 23 tests (plan-named regression guard)
```

`python -m unittest discover -s tests -p 'test_*.py'` was also run; see **Deviations** below — it surfaces a pre-existing, unrelated cross-test contamination issue, not a regression from this plan. See `.planning/phases/02-process-lifecycle-port/deferred-items.md` for the full analysis.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 3 - Blocking] `CodexProcess` does not exist — the plan's class name was wrong**

- **Found during:** Task 1, writing `test_codexrun_process_tree.py`
- **Issue:** The plan's `<action>`/`<behavior>` text names the class `CodexProcess`, but `codexrun.py`'s app-server-spawning class (the one at L488 `__init__`, matching the described `Popen` call and `close()` kill path) is actually named `AppServerClient`. No `CodexProcess` symbol exists in the file.
- **Fix:** Used `codexrun.AppServerClient` in the test instead of the plan's `CodexProcess`. All described line numbers, kwargs, and kill-path behavior matched the plan exactly once the correct class name was used — only the name was wrong.
- **Files modified:** `tests/test_codexrun_process_tree.py`
- **Commit:** `7c09aee`

**2. [Rule 1 - Bug] `requirements mark-complete PROC-03` over-marked a shared requirement**

- **Found during:** post-execution state updates
- **Issue:** PROC-03 is also carried by `02-02-PLAN.md`'s frontmatter (`requirements: [PROC-03]`, targeting `supervisor.py`/`mailhub_runtime.py` — the requirement text itself says "for every process spawn path"). Running `requirements mark-complete PROC-03` after only 02-01 marked it fully `[x]` Complete in both the checkbox and traceability table, which is wrong until 02-02 also lands.
- **Fix:** Reverted `REQUIREMENTS.md`'s PROC-03 checkbox to `[ ]` and its traceability row to "In Progress (02-01 of 2 plans done)", with a parenthetical note on the checkbox line recording what 02-01 completed.
- **Files modified:** `.planning/REQUIREMENTS.md`
- **Commit:** included in the final `docs(02-01)` commit

### Out-of-scope findings (logged, not fixed)

**Pre-existing `unittest discover` cross-test contamination** — see `.planning/phases/02-process-lifecycle-port/deferred-items.md`. Several test modules bind `orgtree.store.DATA_ROOT` from `ORGTREE_DATA` at import time; running the whole `tests/` directory in one interpreter process races that binding across unrelated files, producing 91 failures / 478 errors unrelated to `codexrun.py`/`antigravityrun.py`. Confirmed pre-existing and out of scope: the failing files don't import either module, and both plan-named regression-guard test files pass cleanly run individually. Logged to `.planning/WINDOWS.md` (entry id 2, kind `deviation`).

## Threat Flags

None — both threat register entries (T-02-01, T-02-02) were mitigated exactly as planned; T-02-03 was an accepted pre-existing race window, unchanged.

## Self-Check: PASSED

- FOUND: engine/backend/orgtree/codexrun.py (modified)
- FOUND: engine/backend/orgtree/antigravityrun.py (modified)
- FOUND: tests/test_codexrun_process_tree.py
- FOUND: tests/test_antigravityrun_process_tree.py
- FOUND commit 7c09aee
- FOUND commit d505ae8
- FOUND commit d1d9552
- FOUND commit e3ca4ed
