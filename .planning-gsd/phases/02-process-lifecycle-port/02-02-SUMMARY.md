---
phase: 02-process-lifecycle-port
plan: 02
subsystem: process-lifecycle
tags: [subprocess, process-groups, posix, signal, supervisor, mailhub]

requires:
  - phase: 02.1-engine-process-lifetime-posix-adapter
    provides: gitrunner.py's start_new_session/os.killpg pattern, used verbatim here
provides:
  - supervisor.py's 4 Claude-CLI Popen sites spawn under a new POSIX session
  - _wd_kill_tree() (6+ callers via warmpool.py/halt.py) kills the full process
    group via os.killpg(SIGKILL) on POSIX, not a bare proc.kill()
  - mailhub_runtime.py's start() spawns under a new POSIX session
  - mailhub_runtime.py's _reclaim_orphan() kills the full process group of a
    stale pidfile-referenced process on POSIX
affects: [02-03-PLAN.md, any future process-spawn site in supervisor.py or mailhub_runtime.py]

actuals:
  tokens: 2808
  tasks: 2
  commits: 5

tech-stack:
  added: []
  patterns:
    - "gitrunner.py's start_new_session=(os.name != \"nt\") + POSIX
      os.killpg(pid, signal.SIGKILL)/try-except ProcessLookupError pattern,
      now applied at 2 more call sites/chokepoints"

key-files:
  created:
    - tests/test_supervisor_kill_tree.py
    - tests/test_mailhub_orphan_reclaim.py
  modified:
    - engine/backend/orgtree/supervisor.py
    - engine/mailhub_runtime.py

key-decisions:
  - "Both fixes replicate gitrunner.py's proven pattern verbatim rather than
    inventing a new containment approach — same start_new_session kwarg,
    same os.killpg/SIGKILL/ProcessLookupError shape."
  - "_wd_kill_tree() is fixed once at the shared chokepoint (not at each of
    its 6+ callers in warmpool.py/halt.py), so every caller inherits correct
    containment without any caller-side changes."
  - "mailhub_runtime.py's pre-existing except OSError: pass around the POSIX
    branch is kept as-is; the narrower except ProcessLookupError: pass is
    added inside it to match D-06/gitrunner.py's exact shape, per the plan's
    explicit instruction."

patterns-established: []

requirements-completed: [PROC-03]

coverage:
  - id: D1
    description: "supervisor.py's 4 Claude-CLI Popen spawn sites (cache-keepalive
      L12341, main turn L18323, compact/fork L24014, remote-control L24309)
      all pass start_new_session=(os.name != \"nt\")"
    requirement: "PROC-03"
    verification:
      - kind: unit
        ref: "tests/test_supervisor_kill_tree.py#test_kill_tree_terminates_the_forked_grandchild"
        status: pass
    human_judgment: false
  - id: D2
    description: "_wd_kill_tree() kills the full process group via
      os.killpg(pid, signal.SIGKILL) on POSIX instead of a bare proc.kill(),
      fixed once at the shared chokepoint used by 6+ callers"
    requirement: "PROC-03"
    verification:
      - kind: unit
        ref: "tests/test_supervisor_kill_tree.py#test_kill_tree_terminates_the_forked_grandchild"
        status: pass
      - kind: unit
        ref: "tests.test_provider_attempt_and_liveness.SendPhaseOverRealPipes (regression)"
        status: pass
    human_judgment: false
  - id: D3
    description: "mailhub_runtime.py's start() spawns under a new POSIX session;
      _reclaim_orphan() kills the full process group of a stale pidfile pid"
    requirement: "PROC-03"
    verification:
      - kind: unit
        ref: "tests/test_mailhub_orphan_reclaim.py#test_reclaim_orphan_terminates_the_forked_grandchild"
        status: pass
    human_judgment: false

duration: 23min
completed: 2026-09-23
status: complete
---

# Phase 02 Plan 02: Process-Lifecycle Port (supervisor.py + mailhub_runtime.py) Summary

**Closed PROC-03's two remaining containment gaps — supervisor.py's real "stop agent" chokepoint (`_wd_kill_tree()`, 4 spawn sites, 6+ callers) and mailhub_runtime.py's crash-recovery orphan reclaim — both now kill the full POSIX process group via `gitrunner.py`'s proven `start_new_session`/`os.killpg(SIGKILL)` pattern instead of leaving forked grandchildren orphaned.**

## Performance

- **Duration:** ~23 min (first commit to last commit)
- **Started:** 2026-09-23T09:50:23Z
- **Completed:** 2026-09-23T10:13:50Z
- **Tasks:** 2/2 completed
- **Files modified:** 4 (2 source, 2 new test files) + docs (deferred-items.md, WINDOWS.md)

## Accomplishments
- supervisor.py's 4 Claude-CLI `Popen` sites now spawn under a new POSIX session; `_wd_kill_tree()` (the real "stop button" for every live Claude Code turn, called from 6+ sites in `warmpool.py`/`halt.py`) now kills the entire process group with `os.killpg(pid, signal.SIGKILL)` on POSIX instead of a bare `proc.kill()` that left forked grandchildren running
- `mailhub_runtime.py`'s `start()` spawns under a new POSIX session and `_reclaim_orphan()` now kills the full process group of a stale pidfile-referenced process from a crashed prior engine run, instead of a single-process `SIGTERM`
- Both real-process-tree integration tests were run and shown RED against the pre-fix code (grandchild survived), then GREEN after the fix (grandchild dead within 2s), following the TDD RED/GREEN gate

## Task Commits

Each task was committed atomically as a RED/GREEN pair:

1. **Task 1: supervisor.py — 4 spawn sites + `_wd_kill_tree()` POSIX fix**
   - `a94a018` (test) — failing test for `_wd_kill_tree` grandchild containment (RED)
   - `2c8868a` (feat) — fix supervisor.py spawn sites and `_wd_kill_tree` (GREEN)
2. **Task 2: mailhub_runtime.py — spawn + orphan reclaim POSIX fix**
   - `61b49e2` (test) — failing test for mailhub `_reclaim_orphan` grandchild containment (RED)
   - `c2c4538` (feat) — fix mailhub_runtime.py spawn + orphan reclaim (GREEN)

**Deferred-items/ledger docs:** `18eae93` (docs) — logs 2 pre-existing, out-of-scope failures found during regression verification (not fixed, see below)

_Both tasks used the RED/GREEN TDD cycle as `tdd="true"` requires — no REFACTOR commit was needed (each fix is a minimal, already-isolated change)._

## Files Created/Modified
- `engine/backend/orgtree/supervisor.py` — `import signal`; 4 `Popen(...)` sites gained `start_new_session=(os.name != "nt")`; `_wd_kill_tree()`'s POSIX branch swapped `proc.kill()` for `os.killpg(proc.pid, signal.SIGKILL)` in `try/except ProcessLookupError`
- `engine/mailhub_runtime.py` — `import signal`; `start()`'s `Popen(...)` gained `start_new_session=(os.name != "nt")`; `_reclaim_orphan()`'s POSIX branch swapped `os.kill(pid, 15)` for `os.killpg(pid, signal.SIGKILL)` in `try/except ProcessLookupError` (inside the existing outer `except OSError: pass`)
- `tests/test_supervisor_kill_tree.py` (new) — spawns a real process-group child+grandchild, calls `sup._wd_kill_tree()`, asserts the grandchild is dead within 2s
- `tests/test_mailhub_orphan_reclaim.py` (new) — spawns a real process-group child+grandchild, writes a `hub.pid`-shaped file, calls `MailhubRuntime._reclaim_orphan()` against a minimal `.data_dir` stand-in, asserts the grandchild is dead within 2s
- `.planning/phases/02-process-lifecycle-port/deferred-items.md` (new) — documents 2 pre-existing, out-of-scope failures found during verification (below)
- `.planning/WINDOWS.md` — 2 new `deviation` entries for the same

## Decisions Made
- Followed the plan's exact prescribed diff shape at every site (verified line-for-line against the plan's `<key_links>` and `<action>` blocks before editing) — no deviation from the specified pattern.
- Kept `_wd_kill_tree()`'s `nt` branch, `proc.wait(timeout=5)`, and all 6+ callers untouched; kept `mailhub_runtime.py`'s `stop()` and `nt` branch untouched — both confirmed via `git diff` showing zero changes outside the named functions.
- One test file per task, structured as a single test that is RED against the pre-fix code and GREEN after the fix (not two permanently-coexisting tests), matching the plan's singular acceptance-criterion wording ("asserts ... dead ... and was shown alive against the pre-fix code").

## Deviations from Plan

None in the fix itself — `supervisor.py` and `mailhub_runtime.py` diffs match the plan's `<action>`/`<key_links>` byte-for-byte outside the specified changes.

### Out-of-scope findings (documented, not fixed)

**1. [Scope boundary] `tests/test_mailhub_runtime.py` — 6 pre-existing failures**
- **Found during:** Task 2 regression check
- **Issue:** `engine/mailhub` git submodule is not checked out in this worktree (`git submodule status` shows `-6e856ee...`), so the real `python -m mailhub.serve` child process this suite spawns can't start (exits code 1)
- **Confirmed unrelated:** re-ran the same suite against the unmodified `engine/mailhub_runtime.py` (via `git checkout HEAD --`) — identical 8 passed/6 FAILED
- **Not fixed:** submodule initialization is an environment/worktree setup action outside this plan's scope
- **Documented in:** `.planning/phases/02-process-lifecycle-port/deferred-items.md`, `.planning/WINDOWS.md`

**2. [Scope boundary] `python -m unittest discover -s tests -p 'test_*.py'` — pre-existing, repo-wide, 91 failures/478 errors**
- **Found during:** Task 2 full-suite verification (plan's `<verification>` step 3)
- **Issue:** cross-module `orgtree.store.DATA_ROOT` binding-order dependency plus Windows-path-shaped assertions failing on this macOS run
- **Confirmed unrelated:** re-ran discovery with this plan's 2 new test files temporarily removed — identical 91 failures/478 errors/25 skipped
- **Not fixed:** repo-wide test-suite structural issue outside this plan's 2-file scope (PROC-03 process-tree containment)
- **Documented in:** `.planning/phases/02-process-lifecycle-port/deferred-items.md`, `.planning/WINDOWS.md`
- **Targeted verification used instead:** both new tests individually (pass), plus `tests.test_provider_attempt_and_liveness.SendPhaseOverRealPipes` (pass) — the regression suite named in the plan's `<verification>` as most relevant to supervisor.py's spawn/kill machinery

---

**Total deviations:** 0 in the fix itself; 2 pre-existing out-of-scope findings documented (not auto-fixed, not in scope)

## Known Stubs

None.

## Threat Flags

None — both fixes are the `mitigate` dispositions already named in the plan's `<threat_model>` (T-02-04, T-02-05); T-02-06 (`accept`) is unchanged by this plan.


## Self-Check: PASSED

All created/modified files confirmed present on disk; all 5 commit hashes (`a94a018`, `2c8868a`, `61b49e2`, `c2c4538`, `18eae93`) confirmed present in `git log`.
