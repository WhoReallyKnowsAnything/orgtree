---
phase: 02-process-lifecycle-port
plan: 03
subsystem: testing
tags: [python, unittest, mock, precedence, providers, supervisor, macos]

requires:
  - phase: "02.1"
    provides: "PosixTree adapter and liveness.py's os.kill(pid, 0) POSIX branch, already live-verified this session against real macOS process states"
provides:
  - "AntigravityPathResolutionTests: mocked-path proof that providers.antigravity_path() resolves env > install > PATH, matching codex_path()'s precedence shape"
  - "ClaudeInstallStateResolutionTests: mocked-path proof that supervisor.claude_install_state(force=True) resolves the identical env > pin > PATH order"
  - "Reconfirmed liveness.py's os.kill(pid, 0) POSIX branch against real macOS process states (test_3e_a_live_process_is_alive_and_a_reaped_one_is_dead)"
affects: ["05-end-to-end-verification-test-coverage"]

actuals:
  tokens: 1460
  tasks: 2
  commits: 1

tech-stack:
  added: []
  patterns:
    - "unittest.mock.patch.dict/patch.object as inline context managers per test method (not class-level monkeypatch reassignment) to guarantee stdlib-wide patches (os.path.exists, shutil.which) restore even on assertion failure"
    - "Precedence-under-conflict testing: each branch's test leaves lower-priority signals ALSO resolvable (e.g. env test also sets install+path) to prove the winning branch beats real competition, not just that it works when it's the only option"

key-files:
  created: [tests/test_provider_path_resolution.py]
  modified: []

key-decisions:
  - "No RED/GREEN split despite tdd=\"true\": this is a verification-only task (RESEARCH.md Critical Finding 1 — no new resolver code needed), so there is no production-code GREEN phase to pair with a RED. A single test(...) commit is correct; forcing an artificial failing-test-first step would require deliberately breaking providers.py/supervisor.py, which the plan's acceptance criteria explicitly forbid."
  - "Used unittest.mock.patch.dict/patch.object context managers instead of test_providers_force.py's direct-attribute-reassignment-in-tearDown convention for os.path.exists and shutil.which specifically, since those are stdlib-shared across the whole test process — a context manager restores them even if an assertion fails mid-test, where manual tearDown reassignment would still restore but only makes prior failure diagnostics less confusing (context manager is the stricter guarantee for shared-global patches)."
  - "providers._antigravity_install_path is still monkeypatched via direct module-attribute reassignment inside patch.object (project-owned function, low blast radius) — same shape test_providers_force.py already uses for providers.codex_status/antigravity_status."

patterns-established:
  - "Precedence-order test shape: assert both the returned VALUE and the returned SOURCE TAG per branch, with competing lower-priority signals also live, so the test proves ORDER rather than mere non-None output."

requirements-completed: [PROC-01, PROC-02]

coverage:
  - id: D1
    description: "providers.antigravity_path() proven to resolve ORGTREE_ANTIGRAVITY env override > installer's private location > PATH `agy`, in that precedence order, under conflicting-signal conditions"
    requirement: "PROC-01"
    verification:
      - kind: unit
        ref: "tests/test_provider_path_resolution.py::AntigravityPathResolutionTests"
        status: pass
    human_judgment: false
  - id: D2
    description: "supervisor.claude_install_state(force=True) proven to resolve ORGTREE_CLAUDE env override > private data-root pin > PATH `claude`, in that identical precedence order"
    requirement: "PROC-01"
    verification:
      - kind: unit
        ref: "tests/test_provider_path_resolution.py::ClaudeInstallStateResolutionTests"
        status: pass
    human_judgment: false
  - id: D3
    description: "liveness.py's os.kill(pid, 0) POSIX branch reconfirmed against real macOS process states (live process alive, reaped process dead, non-existent target unknown) on this Darwin 27.2.0 arm64 machine"
    requirement: "PROC-02"
    verification:
      - kind: unit
        ref: "tests/test_provider_attempt_and_liveness.py::LivenessRecord::test_3e_a_live_process_is_alive_and_a_reaped_one_is_dead"
        status: pass
    human_judgment: false
  - id: D4
    description: "Antigravity agent-picker end-to-end launch via the real installer path — the one PROC-01 behavior 02-VALIDATION.md's Manual-Only Verifications table names as unreachable from this dev machine"
    verification: []
    human_judgment: true
    rationale: "Antigravity is not installed on this dev machine (RESEARCH.md's live `which agy` probe: not found). The resolution LOGIC is fully proven by D1's mocked-path unit tests; only the real-binary launch through it needs a machine that has Antigravity installed. Deferred as a known, non-blocking gap per the plan's <human-check> block — not a regression, not a scope gap in PROC-01 itself."

duration: 12min
completed: 2026-09-23
status: complete
---

# Phase 02 Plan 03: Provider Path Resolution Verification Summary

**Mocked-path unit tests proving `antigravity_path()` and `supervisor.claude_install_state()` share `codex_path()`'s env > pin/install > PATH precedence, plus a reconfirmed liveness.py POSIX branch — no production code touched.**

## Performance

- **Duration:** 12 min
- **Started:** 2026-09-23T09:38:00Z
- **Completed:** 2026-09-23T09:50:46Z
- **Tasks:** 2 completed
- **Files modified:** 1 (new)

## Accomplishments
- Wrote `tests/test_provider_path_resolution.py` with 8 test methods (4 per resolver) proving `antigravity_path()` and `supervisor.claude_install_state(force=True)` resolve identical env > pin/install > PATH precedence, each branch tested with lower-priority signals also live to prove the winning branch beats real competition — not just that some path comes back.
- Reconfirmed `tests/test_provider_attempt_and_liveness.py`'s `test_3e_a_live_process_is_alive_and_a_reaped_one_is_dead` and the full 23-test `LivenessRecord`/attempt-phase suite pass on this macOS (Darwin 27.2.0 arm64) environment.
- Both PROC-01 and PROC-02 closed as verification-only per RESEARCH.md — `providers.py`, `supervisor.py`, and `liveness.py` were confirmed correct with zero production-code changes.

## Task Commits

Each task was committed atomically:

1. **Task 1: Mocked-path resolution tests for antigravity_path() and supervisor.claude_install_state()** - `ff1662d` (test)
2. **Task 2: Verify liveness.py's os.kill(pid, 0) POSIX branch against real macOS behavior** - no commit (no files changed; existing test suite re-run only, confirmed passing)

**Plan metadata:** (pending — this SUMMARY's own commit)

_Note: Task 1 has a single `test(...)` commit, not a test→feat pair — see Deviations for why._

## Files Created/Modified
- `tests/test_provider_path_resolution.py` - New file. `AntigravityPathResolutionTests` (4 tests) and `ClaudeInstallStateResolutionTests` (4 tests), following `tests/test_providers_force.py`'s tempdir/`ORGTREE_DATA`/`sys.path.insert` import convention.

## Decisions Made
- No RED/GREEN commit split for Task 1 despite `tdd="true"`: this plan is verification-only (RESEARCH.md Critical Finding 1), so there is no implementation step to pair a RED phase with — the acceptance criteria explicitly forbid changing `providers.py`/`supervisor.py`. A single `test(...)` commit is the correct shape; see `## TDD Gate Compliance` below.
- Used `unittest.mock.patch.dict`/`patch.object` as inline context managers rather than test_providers_force.py's setUp/tearDown attribute-reassignment convention, specifically for `os.path.exists` and `shutil.which` — both stdlib-shared across the whole test process, so a context manager's guaranteed restore-on-exception is the stricter safety property for a global patch. `providers._antigravity_install_path` still uses the project's existing direct-reassignment style since it is a project-owned, low-blast-radius function.

## Deviations from Plan

None - plan executed exactly as written. Task 2 made no code changes, as specified; the SUMMARY documents the zombie-hazard finding below per the plan's `<action>` instruction.

## TDD Gate Compliance

Task 1 carries `tdd="true"` but this plan is verification-only end to end (no `providers.py`/`supervisor.py` changes are permitted by its own acceptance criteria), so there is no GREEN implementation step to gate against. The single `test(02-03): mocked-path resolution tests...` commit (`ff1662d`) both wrote and green-lit the tests in one step — the tests passing on first run IS the verification this task exists to produce, not a RED-phase violation, since the resolvers under test were never expected to change.

## Issues Encountered

None. `python -m unittest tests.test_provider_path_resolution -v` and `python -m unittest tests.test_provider_attempt_and_liveness -v` both exit 0 (8/8 and 23/23 respectively) on the first run.

## Zombie-Hazard Finding (Task 2, accepted/pre-existing)

Per RESEARCH.md Pitfall 1, independently confirmed live this session: an exited-but-unreaped child still answers `os.kill(pid, 0)` as `"still-active"` — `_observe_pid()`'s POSIX branch (`liveness.py` L190-198) cannot distinguish a genuine live process from a zombie by this probe alone. This is a pre-existing, accepted characteristic of the POSIX branch, not a regression and not a gap PROC-02 asks to close: the requirement only asks the branch to match real macOS `os.kill(pid, 0)` behavior, which it does exactly (the OS itself makes the same non-distinction). No code change made.

## Checkpoint: Manual-Only Verification (non-blocking)

02-VALIDATION.md's Manual-Only Verifications table names one PROC-01 behavior this task's mocked tests cannot reach: installing Antigravity on a real macOS machine, running the app, and confirming the agent picker resolves and launches the Antigravity provider via its real `~/.local/bin/agy` install with no manual path override needed.

- **Why deferred:** Antigravity is not installed on this dev machine (RESEARCH.md's live `which agy` probe: not found).
- **What is already proven:** `antigravity_path()`'s resolution LOGIC — env > install > PATH precedence — is fully proven by this plan's mocked-path unit tests (D1 above). Only the real-binary end-to-end launch needs a machine with Antigravity installed.
- **Blocking status:** Non-blocking, per the plan's own `<human-check>` guidance. Recorded here as a checkpoint item for whoever next has access to a machine with Antigravity installed, not as an open defect in this phase.

## User Setup Required

None.

## Self-Check: PASSED

- FOUND: tests/test_provider_path_resolution.py
- FOUND: ff1662d (git log --oneline --all | grep ff1662d)
