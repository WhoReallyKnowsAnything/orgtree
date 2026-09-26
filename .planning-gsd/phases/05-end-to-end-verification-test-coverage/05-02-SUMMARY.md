---
phase: 05-end-to-end-verification-test-coverage
plan: 02
subsystem: testing
tags: [posix, acl, permissions, process-lifetime, service-host, macos, windows-parity]

# Dependency graph
requires: []
provides:
  - Real POSIX (chmod 0600 + O_CREAT|O_EXCL + stat/uid verification) descriptor-protection
    implementation in engine/service_host.py, replacing the prior unconditional
    Windows-only no-op/raise
  - 6 macOS-only companion tests in tests/test_service_host.py proving the POSIX
    descriptor-protection fail-closed contract
  - 2 macOS-only companion tests in tests/test_claude_pipe_lifecycle.py proving the idle
    watchdog / pipe-expiry process-tree kill already works on POSIX
  - Source-verified, corrected dispositions for every Windows-only test skip in
    tests/test_process_lifetime.py, tests/test_startup_progress.py, tests/test_startup_readiness.py
affects: [phase-5-remaining-plans, any-future-audit-of-VER-02]

# Actuals (#2632)
actuals:
  tokens: 3515
  tasks: 3
  commits: 2

tech-stack:
  added: []
  patterns:
    - "POSIX branch mirrors Windows contract exactly: chmod 0600 in place of icacls inheritance-strip, os.open(O_CREAT|O_EXCL|O_WRONLY) in place of CreateFileW+SDDL, os.stat().st_mode/st_uid in place of Get-Acl"
    - "macOS-only companion tests are named with an _on_posix suffix mirroring their Windows-only sibling's name, so pairing is discoverable by name alone"

key-files:
  created: []
  modified:
    - engine/service_host.py
    - tests/test_service_host.py
    - tests/test_claude_pipe_lifecycle.py
    - tests/test_startup_progress.py
    - tests/test_startup_readiness.py

key-decisions:
  - "Task 3's two test_claude_pipe_lifecycle.py companions were written as genuinely PASSING tests, not @unittest.expectedFailure as the plan specified — source inspection of engine/backend/orgtree/supervisor.py showed _wd_kill_tree already uses os.killpg(SIGKILL) and _run_one_turn already sets start_new_session=(os.name != \"nt\") on POSIX, so the process-tree kill this test exercises already works today; the codexrun.py/antigravityrun.py gap the plan cited is a different code path this fixture-based test never touches"
  - "tests/test_startup_readiness.py's Windows-only class gate was removed entirely (not just its reason string strengthened) after both of its tests were verified passing on POSIX unmodified"
  - "tests/test_process_lifetime.py was left untouched — it has no Windows-only gate at all, and all 8 of its tests (exercising engine/process_lifetime.py's already-POSIX-aware guardian) already pass on this machine"
  - "tests/test_startup_progress.py's BootStartupTests gate was kept but its reason string was rewritten to name the actually-verified gap (RootLock's fcntl release timing after a failed-start guardian teardown), not the plan's inaccurate 'guardian design does not exist' premise"

requirements-completed: [VER-02]

coverage:
  - id: D1
    description: "engine/service_host.py has a real POSIX descriptor-protection implementation (chmod 0600, O_CREAT|O_EXCL, stat/uid verification), not a no-op"
    requirement: "VER-02"
    verification:
      - kind: unit
        ref: "tests/test_service_host.py::ServiceHostUnitTests (23 tests, 6 new POSIX-only, 2 unaffected regression tests)"
        status: pass
    human_judgment: false
  - id: D2
    description: "6 macOS-only companion tests exist in tests/test_service_host.py and pass today, paired by name with their Windows-only siblings"
    requirement: "VER-02"
    verification:
      - kind: unit
        ref: "tests/test_service_host.py::ServiceHostUnitTests::test_*_on_posix (6 methods)"
        status: pass
    human_judgment: false
  - id: D3
    description: "2 macOS-only companion tests exist in tests/test_claude_pipe_lifecycle.py for the idle-watchdog / pipe-expiry Windows-only originals; verified passing (not expected-fail) after source investigation showed the underlying process-tree kill already works on POSIX"
    requirement: "VER-02"
    verification:
      - kind: unit
        ref: "tests/test_claude_pipe_lifecycle.py::ClaudePipeLifecycleTests::test_idle_watchdog_ends_launcher_and_child_then_returns_queued_mail_on_posix, test_expiry_releases_reader_even_when_a_child_keeps_the_pipe_open_on_posix"
        status: pass
    human_judgment: false
  - id: D4
    description: "Every remaining Windows-only skip (test_process_lifetime.py, test_startup_progress.py, test_startup_readiness.py) has an explicit, source-verified disposition"
    requirement: "VER-02"
    verification:
      - kind: unit
        ref: "tests/test_process_lifetime.py (no gate, 8/8 pass), tests/test_startup_progress.py::BootStartupTests (gate kept, reason rewritten), tests/test_startup_readiness.py::StartupReadinessTests (gate removed, 2/2 pass)"
        status: pass
    human_judgment: false

# Metrics
duration: 32min
completed: 2026-09-23
status: complete
---

# Phase 5 Plan 2: POSIX Descriptor Protection + Windows-Only Skip Dispositions Summary

**Implemented the missing POSIX branch of service_host.py's descriptor-protection chain (previously an unconditional no-op/raise) and, while auditing every remaining Windows-only test skip for VER-02, found that most of the "blocked on Phase 2/05-01" gaps the plan predicted don't actually exist in current source — the process-tree guardian and its kill path already work on POSIX today.**

## Performance

- **Duration:** 32 min
- **Tasks:** 3/3 completed
- **Files modified:** 5

## Accomplishments

- `engine/service_host.py`'s descriptor-protection chain (`_current_user_sid`, `restrict_descriptor_acl`, `create_protected_exclusive`, `verify_restricted_acl`, `write_descriptor`) now has a real POSIX implementation mirroring the Windows SDDL/icacls contract exactly (chmod 0600, O_CREAT|O_EXCL, stat/uid verification), instead of returning `False`/raising unconditionally.
- 6 new macOS-only tests added to `tests/test_service_host.py`, each paired by name (`_on_posix` suffix) with its Windows-only sibling; all pass on this machine.
- 2 new macOS-only tests added to `tests/test_claude_pipe_lifecycle.py` for the idle-watchdog and pipe-expiry Windows-only originals — written as genuinely passing tests after source investigation proved the plan's stated blocker (missing `start_new_session` in codexrun.py/antigravityrun.py) doesn't apply to this code path, which already goes through `supervisor.py`'s already-POSIX-aware `_wd_kill_tree`/`start_new_session`.
- `tests/test_startup_readiness.py`'s entire Windows-only class gate removed — both its tests pass unmodified on POSIX.
- `tests/test_startup_progress.py`'s `BootStartupTests` gate kept (one of two tests genuinely fails on POSIX), but its reason string rewritten to name the actually-verified gap instead of a stale premise.
- `tests/test_process_lifetime.py` audited and left untouched: it has no Windows-only gate, and all 8 of its tests already pass on POSIX.

## Task Commits

Each task was committed atomically:

1. **Task 1 + Task 2: Real POSIX descriptor-protection + 6 macOS-only tests** - `23b5a75` (feat)
2. **Task 3: Explicit disposition for tests blocked on earlier-phase work** - `f8f3115` (test)

Tasks 1 and 2 were committed together since they touch the same two files (`engine/service_host.py`, `tests/test_service_host.py`) and the verify step for Task 1 is a strict subset of Task 2's.

## Files Created/Modified

- `engine/service_host.py` - Added POSIX branches to `_current_user_sid`, `restrict_descriptor_acl`, `create_protected_exclusive`, `verify_restricted_acl`, `write_descriptor`
- `tests/test_service_host.py` - Added `import stat` and 6 macOS-only companion test methods
- `tests/test_claude_pipe_lifecycle.py` - Added `posix_alive()` helper and 2 macOS-only companion test methods (passing, not expected-fail)
- `tests/test_startup_progress.py` - Strengthened `BootStartupTests`' skip-reason string with the actually-verified gap
- `tests/test_startup_readiness.py` - Removed the stale Windows-only class gate (both tests pass on POSIX)

## Decisions Made

- Kept the plan's line-for-line POSIX implementation choices for Task 1/2 (chmod 0600, O_CREAT|O_EXCL, stat/uid check) — these matched the plan exactly and needed no deviation.
- For Task 3, investigated each "blocked on Phase 2/05-01" claim against the actual current source before writing any disposition, per VER-02's own truth #2 ("explicit, source-grounded disposition"). Three of the plan's four premises turned out to be stale or inaccurate for the current codebase state (see Deviations below); only `test_startup_progress.py`'s `BootStartupTests` genuinely still has an open gap, and its reason string now names the real one.

## Deviations from Plan

### Auto-fixed Issues (Rule 1 — plan premise did not match verified source)

**1. [Rule 1] `test_claude_pipe_lifecycle.py` companions written as passing tests, not `@unittest.expectedFailure`**
- **Found during:** Task 3
- **Issue:** The plan instructed marking the 2 new companions `@unittest.expectedFailure`, citing "Phase 2 PROC-03 — codexrun.py/antigravityrun.py's process-tree kill is missing `start_new_session`" as the blocker. Reading `engine/backend/orgtree/supervisor.py` directly showed `_wd_kill_tree` (line 29481) already uses `os.killpg(proc.pid, signal.SIGKILL)` on POSIX, and `_run_one_turn`'s own `subprocess.Popen` (line 18323) already sets `start_new_session=(os.name != "nt")`. This test file exercises that code path directly via a mocked `_build_cmd`, never touching codexrun.py/antigravityrun.py at all.
- **Fix:** Wrote the two companions (`test_idle_watchdog_ends_launcher_and_child_then_returns_queued_mail_on_posix`, `test_expiry_releases_reader_even_when_a_child_keeps_the_pipe_open_on_posix`) as plain tests with a `posix_alive()` helper (signal-0 liveness check) in place of `windows_alive()`. Ran them individually and as part of the full suite; both pass.
- **Files modified:** tests/test_claude_pipe_lifecycle.py
- **Commit:** f8f3115

**2. [Rule 1] `test_startup_readiness.py`'s Windows-only gate removed, not just its reason string strengthened**
- **Found during:** Task 3
- **Issue:** The plan said to leave this file's gate as-is and only strengthen its skip-reason text ("INERT: real guardian/readiness budget requires Windows"). Probed the class with the gate temporarily disabled: both `test_positive_control_old_lifespan_order_misses_budget` and `test_ready_inside_budget_while_repair_is_blocked_and_writes_wait` passed unmodified in 21s (real 500-node/20k-entry engine, readiness inside budget while a repair sweep is held). Neither test imports or exercises anything Windows-specific — `tests/startup_engine_probe.py` has no `os.name`/`creationflags`/`msvcrt` references at all.
- **Fix:** Removed the `@unittest.skipUnless(os.name == "nt", ...)` decorator entirely rather than leave a stale "Windows-only" label on a test that demonstrably runs cross-platform — this is the "passing macOS counterpart" disposition VER-02's truths call for, not a strengthened-but-still-false skip string.
- **Files modified:** tests/test_startup_readiness.py
- **Commit:** f8f3115

**3. [Rule 1] `test_process_lifetime.py` left untouched — plan's premise about it doesn't hold**
- **Found during:** Task 3
- **Issue:** The plan's action text claimed this file needs its skip-reason strengthened, citing `engine/process_lifetime.py::arm_process_lifetime` as unconditionally raising `RuntimeError` on non-`nt`. Reading `engine/process_lifetime.py` directly showed `arm_process_lifetime` already branches on `os.name` for its environment allowlist (line 41-44) and sets `start_new_session=os.name != "nt"` (line 51-52) — it is already implemented for POSIX. Searching the test file for any `skip`/`Skip`/`os.name != 'nt'`-gated test found none: `test_process_lifetime.py` has no Windows-only gate anywhere. Ran the full file: all 8 tests pass on this machine today.
- **Fix:** No changes made to this file — there is no skip-reason string to strengthen, and the tests already exercise the real POSIX guardian successfully.
- **Files modified:** none
- **Commit:** n/a (no change)

**4. [Rule 1] `test_startup_progress.py`'s skip-reason rewritten to the actually-verified gap**
- **Found during:** Task 3
- **Issue:** The plan's premise ("guardian requires Windows") would have been left unstrengthened-but-still-inaccurate if applied literally, since one of the two `BootStartupTests` methods (`test_progress_allows_boot_beyond_fixed_deadline`) demonstrably passes on POSIX when probed. The other (`test_silence_and_duplicate_progress_fail_with_proven_release`) genuinely fails: after a failed boot, `engine/process_lifetime.py::RootLock`'s `fcntl.flock` is not released promptly enough for the test's no-retry re-acquire assumption, raising `BlockingIOError`/`RuntimeError`.
- **Fix:** Kept the class-level Windows-only gate (both tests must stay together since they share the same class-level gate), but rewrote the reason string to name this specific, verified gap instead of the plan's inaccurate "guardian requires Windows" framing. Fixing the underlying `RootLock` timing race is out of this plan's file scope (`engine/process_lifetime.py` isn't in this plan's `files_modified`) and belongs to whichever plan owns that module.
- **Files modified:** tests/test_startup_progress.py
- **Commit:** f8f3115

## Threat Flags

None — all changes are within the threat model already declared in the plan (T-05-05, T-05-06, T-05-07), and no new trust boundaries were introduced.

## Self-Check: PASSED

- FOUND: engine/service_host.py (POSIX branches present, verified via git diff)
- FOUND: tests/test_service_host.py (6 new `_on_posix` methods present)
- FOUND: tests/test_claude_pipe_lifecycle.py (`posix_alive` + 2 new `_on_posix` methods present)
- FOUND: tests/test_startup_progress.py (strengthened skip-reason present)
- FOUND: tests/test_startup_readiness.py (gate removed, class runs unconditionally)
- FOUND: commit 23b5a75 (`git log --oneline --all | grep 23b5a75`)
- FOUND: commit f8f3115 (`git log --oneline --all | grep f8f3115`)
- Full consolidated run: `tests.test_service_host tests.test_claude_pipe_lifecycle tests.test_process_lifetime tests.test_startup_progress tests.test_startup_readiness` → 57 tests, OK, skipped=12 (all 12 skips are genuine Windows-only originals with paired POSIX companions or documented dispositions)
