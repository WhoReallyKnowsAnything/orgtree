---
gsd_state_version: "1.0"
current_phase: 1
current_phase_name: Packaging & Runtime Foundation
current_plan: Not started
status: planning
stopped_at: Phase 2 complete, ready to plan Phase 1
last_updated: "2026-09-23T10:36:59.507Z"
last_activity: 2026-09-23
last_activity_desc: Phase 2 complete, transitioned to Phase 1
state_head: b53c8c544b43eb536e54502a7fc31cc1c56ba86c
progress:
  total_phases: 6
  completed_phases: 1
  total_plans: 13
  completed_plans: 7
  percent: 17
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-09-17)

**Core value:** The app runs and works correctly on macOS: build, launch, spawn agents, and manage their work end-to-end — matching what the Windows build already does.
**Current focus:** Phase 02

## Current Position

Phase: 1 — Packaging & Runtime Foundation
Current Plan: Not started
Total Plans in Phase: 2
Status: Ready to plan
Last activity: 2026-09-23 — Phase 2 complete, transitioned to Phase 1

Progress: [██░░░░░░░░] 17%

## Performance Metrics

Total plans completed: 0
Average duration per plan: N/A
Recent trend: N/A (no plans executed yet)
**Per-Plan Metrics:**

| Plan | Duration | Tasks | Files |
|------|----------|-------|-------|
| Phase 02.1 P01 | 50min | 3 tasks | 4 files |
| Phase 02 P01 | ~35 min | 2 tasks | 4 files |
| Phase 02 P02 | 23min | 2 tasks | 6 files |
| Phase 02 P03 | 12min | 2 tasks | 1 files |

## Accumulated Context

**Decisions:** See .planning/PROJECT.md Key Decisions. Roadmap-level decision: sequence packaging (Phase 1) before process lifecycle (Phase 2) before autostart (Phase 3), since nothing is testable on real hardware until a launchable `.app` exists, and the LaunchAgent (Phase 3) invokes the engine ported in Phase 2. UI/OS parity (Phase 4) is isolated and deferred until after the harder plumbing; verification (Phase 5) closes the milestone.

**Pending Todos:** None captured yet.

**Blockers/Concerns:**

- Phase 1: AMFI enforcement on Apple Silicon blocks unsigned binaries even offline ("app is damaged" dialog) — mitigated by ad-hoc signing (`mac.identity: "-"`), but must be verified on real Apple Silicon hardware early (PKG-03).
- Phase 1: nested unsigned binaries inside the `.app` bundle can re-trigger the AMFI failure if ad-hoc signing doesn't cover the full bundle, not just the top-level `.app`.
- Phase 3: `openAtLogin`/LaunchAgent reliability under an unsigned build is a documented caveat, not yet a tested outcome — verify by hand.
- ~~Owned by Phase 2.1 (PROC-04): `arm_process_lifetime` raises unconditionally when `os.name != "nt"`~~ — RESOLVED 2026-09-17 by 02.1-01-PLAN.md (commits `30dec80`/`fbc9fc4`/`4af4984` on `session/execute-phase-2-1-engine-process-lifetim`): `PosixTree` adapter implemented, `engine/launch.py::main()` self-setpgids, `LifetimeTests`/`LaunchRefusalTests` un-skipped and passing on macOS. Residual gap (NOT resolved): a descendant in its own escaped process group is only provably swept when the guardian's teardown runs while the engine is still alive — see `02.1-01-SUMMARY.md` Threat Flags and `.planning/WINDOWS.md` entry #1.
- ~~RESOLVED 2026-09-23 (PKG-03): the packaged `.app` failed `codesign --verify --deep --strict` after launch~~ — CLOSED. Cause was two independent code paths writing `__pycache__/*.cpython-313.pyc` into the signed `Contents/Resources/engine/` tree at launch: the main engine spawn (`apps/desktop/main/engine.ts`) and its guardian watchdog re-exec (`engine/process_lifetime.py::arm_process_lifetime`, whose deliberately strict allowlisted env for the child silently dropped `PYTHONDONTWRITEBYTECODE` even after the first fix). Fix: `PYTHONDONTWRITEBYTECODE=1` set explicitly in both spawn sites (commits `6175497`, `d9021bd`); `codesign -dv` verification-trap checks replaced with `--verify --deep --strict` in the plan and `docs/macos-first-launch.md` (commit `fd5f9fb`). Task 4 (`checkpoint:human-verify`) run by the user on real Apple Silicon 2026-09-23: PASS — in-place launch dialog-free (one unrelated macOS TCC file-access privacy prompt appeared, not a signing issue — see `01-03-SUMMARY.md` Deviations), seal intact after quitting, quarantined copy showed the recoverable "cannot be verified" dialog, not "is damaged". Intel/x64 hardware verification not available this session; documented follow-up per the plan's own allowance.

## Session Continuity

**Stopped at:** Phase 2 complete, ready to plan Phase 1

Last session: 2026-09-23T10:19:31.846Z
Last completed: Phase 1 closed — PKG-03 code fix (both spawn sites), verification-trap doc/plan cleanup, and Task 4's real-hardware human checkpoint (PASS), all this session. Phase 02.1 execution and Phase 1 Tasks 1-3 completed 2026-09-17.
Resume file: None

## Decisions

- [Phase 1]: PosixTree adapter mirrors WindowsTree's exact public surface; kqueue EVFILT_PROC/NOTE_EXIT for exit detection, ps-based enumerate-then-killpg sweep for teardown
- [Phase 1]: Escaped-process-group descendants are only provably swept when the guardian's teardown runs while the engine is still alive (parent-death/guardian-crash); the engine-dies-first case has a documented residual gap (see 02.1-01-SUMMARY.md Threat Flags)
- [Phase 02]: Mirrored gitrunner.py's proven _stop() pattern in codexrun.py close() and antigravityrun.py kill_tree(): start_new_session on spawn, os.killpg(pid, SIGKILL) on the POSIX kill path; Windows taskkill branches untouched
- [Phase 02]: supervisor.py + mailhub_runtime.py POSIX process-tree containment: applied gitrunner.py's start_new_session/os.killpg pattern verbatim at both fix sites, fixed _wd_kill_tree() once at the shared chokepoint
- [Phase 02]: 02-03: verification-only — no RED/GREEN split; a single test(...) commit both wrote and green-lit the precedence-order proof since no production code changes were permitted
