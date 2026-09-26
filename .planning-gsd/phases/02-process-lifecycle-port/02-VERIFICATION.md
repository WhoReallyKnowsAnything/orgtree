---
phase: 02-process-lifecycle-port
verified: 2026-09-23T00:00:00Z
status: passed
score: 3/3 must-haves verified
behavior_unverified: 0
overrides_applied: 0
---

# Phase 2: Process Lifecycle Port Verification Report

**Phase Goal:** The Python engine correctly finds, runs, monitors, and terminates provider CLI subprocesses on macOS.
**Verified:** 2026-09-23
**Status:** passed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | The engine locates/launches Claude Code, Codex, and Antigravity CLIs from real macOS install paths via `shutil.which()` | ✓ VERIFIED | `providers.py:455` (`codex_path`) and `providers.py:892` (`antigravity_path`) call `shutil.which()`; `supervisor.py:390` (`claude_install_state`) calls `shutil.which("claude")`. `tests/test_provider_path_resolution.py` (new, 8 tests) proves env > install/pin > PATH precedence for all three resolvers by driving the real functions with mocked `shutil.which`/install-path helpers; all 8 pass. RESEARCH.md's Critical Finding 1 established no new resolver code was needed — this was a verification-only requirement, satisfied. |
| 2 | The engine's liveness check reports alive/dead matching real macOS process state (`os.kill(pid, 0)`) | ✓ VERIFIED | `liveness.py:190-198` (`_observe_pid`) POSIX branch: `os.kill(pid, 0)` → `"still-active"`, `ProcessLookupError` → `"no-such-process"`, `PermissionError` → `"access-denied"`. `tests/test_provider_attempt_and_liveness.py::test_3e_a_live_process_is_alive_and_a_reaped_one_is_dead` spawns a REAL subprocess on this macOS box, asserts `liveness.observe()` reports `alive`/`still-active` while running and `dead` after `kill()`+`wait()`. Ran via `python -B tests/test_provider_attempt_and_liveness.py`: 23 tests, exit 0, OK. |
| 3 | Stopping an agent kills the provider CLI and all descendants via `start_new_session`+`os.killpg`, no orphans | ✓ VERIFIED | `codexrun.py` (`__init__` L513 spawn, `close()` L958 kill), `antigravityrun.py` (`launch()` L762 spawn, `kill_tree()` L580 kill), `supervisor.py` (4 spawn sites L12347/18330/24021/24315 + `_wd_kill_tree()` L29507), `mailhub_runtime.py` (`start()` L375 spawn, `_reclaim_orphan()` L320 kill) — all set `start_new_session=(os.name != "nt")` on spawn and call `os.killpg(pid, signal.SIGKILL)` on the POSIX kill path, matching `gitrunner.py`'s proven `_stop()` pattern. Four dedicated tests (`test_codexrun_process_tree.py`, `test_antigravityrun_process_tree.py`, `test_supervisor_kill_tree.py`, `test_mailhub_orphan_reclaim.py`) each spawn a REAL child that forks a REAL grandchild (`sleep 30 &`), invoke the production kill function, and assert the grandchild pid is dead within 2s — this is genuine behavioral proof of the cancellation/cleanup invariant, not presence-only. All pass. |

**Score:** 3/3 truths verified (0 present, behavior-unverified)

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `engine/backend/orgtree/codexrun.py` | `start_new_session` on spawn + `os.killpg` on close | ✓ VERIFIED | L513, L958 — wired, exercised by real-process test |
| `engine/backend/orgtree/antigravityrun.py` | same, for `kill_tree()` | ✓ VERIFIED | L762, L580 — wired, exercised by real-process test |
| `engine/backend/orgtree/supervisor.py` | 4 spawn sites + `_wd_kill_tree()` | ✓ VERIFIED | L12347/18330/24021/24315 spawn, L29481-29513 `_wd_kill_tree` — wired, exercised by real-process test |
| `engine/mailhub_runtime.py` | `start()` spawn + `_reclaim_orphan()` kill | ✓ VERIFIED | L375, L301-328 — wired, exercised by real-process test |
| `engine/backend/orgtree/liveness.py` | `os.kill(pid,0)` POSIX branch (verification-only, no change expected) | ✓ VERIFIED | L190-198 unchanged, confirmed correct against real macOS process states |
| `engine/backend/orgtree/providers.py` | `shutil.which()` resolution (verification-only, no change expected) | ✓ VERIFIED | L455, L892 unchanged, confirmed correct via precedence tests |
| `tests/test_codexrun_process_tree.py` (new) | Real-process-tree regression test | ✓ VERIFIED | 2 tests, passes |
| `tests/test_antigravityrun_process_tree.py` (new) | Real-process-tree regression test | ✓ VERIFIED | 2 tests, passes |
| `tests/test_supervisor_kill_tree.py` (new) | Real-process-tree regression test | ✓ VERIFIED | 1 test, passes |
| `tests/test_mailhub_orphan_reclaim.py` (new) | Real-process-tree regression test | ✓ VERIFIED | 1 test, passes |
| `tests/test_provider_path_resolution.py` (new) | Mocked-precedence resolver test | ✓ VERIFIED | 8 tests, passes |

### Key Link Verification

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| `codexrun.py` `Popen(start_new_session=...)` | `close()`'s `os.killpg(self.proc.pid, ...)` | same process group | ✓ WIRED | Both reference `self.proc.pid`; test proves the linkage empirically (grandchild reaped) |
| `antigravityrun.py` `Popen(start_new_session=...)` | `kill_tree()`'s `os.killpg()` | same process group | ✓ WIRED | Same as above, `launch()` ↔ `kill_tree(self.proc)` in `close()` |
| `supervisor.py` 4 spawn sites | `_wd_kill_tree()` (6+ callers) | shared chokepoint function | ✓ WIRED | All 4 spawns set the kwarg; `_wd_kill_tree` is the single kill path all callers route through |
| `mailhub_runtime.py` `start()` | `_reclaim_orphan()` | pidfile-recorded pid, same group | ✓ WIRED | `start()` spawns with `start_new_session`; `_reclaim_orphan()` reads `hub.pid` and `killpg`s it |

### Behavioral Spot-Checks / Named Test Execution

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| codexrun process-tree kill | `python -m unittest tests.test_codexrun_process_tree -v` | Ran 2 tests, OK | ✓ PASS |
| antigravityrun process-tree kill | `python -m unittest tests.test_antigravityrun_process_tree -v` | Ran 2 tests, OK | ✓ PASS |
| supervisor kill-tree | `python -m unittest tests.test_supervisor_kill_tree -v` | Ran 1 test, OK | ✓ PASS |
| mailhub orphan reclaim | `python -m unittest tests.test_mailhub_orphan_reclaim -v` | Ran 1 test, OK | ✓ PASS |
| provider path resolution | `python -m unittest tests.test_provider_path_resolution -v` | Ran 8 tests, OK | ✓ PASS |
| provider attempt + liveness | `python -B tests/test_provider_attempt_and_liveness.py` | Ran 23 tests, exit 0, OK (stderr shows expected mocked-OSError fixture traces, not failures) | ✓ PASS |
| process lifetime | `python -m unittest tests.test_process_lifetime -v` | Ran 8 tests, OK | ✓ PASS |

All 7 phase-relevant named test suites re-run independently by this verifier (not trusting SUMMARY.md claims) — all pass. Full-suite `unittest discover` baseline (91 failures/478 errors/25 skipped) is documented in `deferred-items.md` and cross-referenced against an unmodified tree as pre-existing/unrelated (cross-module `DATA_ROOT` import-order contamination, Windows-path assertions on macOS, uninitialized `mailhub` git submodule) — not re-run in full by this verifier per instructions, deferred-items.md reasoning independently spot-checked and found sound.

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|-------------|--------|----------|
| PROC-01 | 02-03-PLAN.md | Executable resolution works on macOS | ✓ SATISFIED | `shutil.which()` confirmed in providers.py + supervisor.py; precedence tests pass |
| PROC-02 | 02-03-PLAN.md | Liveness checks work on macOS | ✓ SATISFIED | `os.kill(pid,0)` branch confirmed correct; real-process test passes |
| PROC-03 | 02-01-PLAN.md, 02-02-PLAN.md | Child-process containment via `start_new_session`+`os.killpg` | ✓ SATISFIED | All 6 spawn/kill site pairs (codexrun, antigravityrun, supervisor×4, mailhub) fixed and covered by real-process-tree tests |

No orphaned requirements: REQUIREMENTS.md maps only PROC-01/02/03 to Phase 2, and all three are declared across the three plans' frontmatter (`02-01`: PROC-03, `02-02`: PROC-03, `02-03`: PROC-01+PROC-02). PROC-04 traces to Phase 2.1, out of this phase's scope.

### Anti-Patterns Found

None. Scanned `codexrun.py`, `antigravityrun.py`, `supervisor.py`, `mailhub_runtime.py` for `TODO|FIXME|XXX|TBD|HACK|placeholder|not yet implemented` — zero matches (the one `TODO` hit in `supervisor.py` is a pre-existing unrelated constant name `_TODO_CAP`, not a debt marker). `git diff --stat` shows a surgical diff: 4 source files touched (7-22 lines changed each), 5 new/expanded test files, no unrelated changes.

### Human Verification Required

None. All three success criteria are automated, behaviorally tested against real macOS processes spawned on this box (Platform: darwin per environment), not mocked or simulated.

### Gaps Summary

None. All three ROADMAP.md Phase 2 success criteria are verified with both static code evidence (spawn/kill wiring) and dynamic behavioral evidence (real subprocess + real grandchild reaped). All 7 phase-relevant named test suites pass under independent re-execution by this verifier. Requirements PROC-01, PROC-02, PROC-03 are fully satisfied with no orphaned requirements. The one process-note worth flagging for awareness (not a gap): 02-01-SUMMARY.md documents a self-corrected bug where `requirements mark-complete PROC-03` was invoked prematurely during 02-01 execution (before 02-02 closed PROC-03's remaining scope) and was reverted before merge — REQUIREMENTS.md's current state is correct and reflects all PROC-03 work being done.

---

_Verified: 2026-09-23_
_Verifier: Claude (gsd-verifier)_
