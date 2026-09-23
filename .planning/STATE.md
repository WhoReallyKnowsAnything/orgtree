---
gsd_state_version: "1.0"
current_phase: 1
current_phase_name: Packaging & Runtime Foundation
status: planning
stopped_at: Completed 02-03-PLAN.md
last_updated: "2026-09-23T09:51:39.734Z"
last_activity: 2026-09-23
last_activity_desc: "Resumed and closed Phase 1: fixed PKG-03 (PYTHONDONTWRITEBYTECODE=1 in both the packaged engine spawn and its guardian subprocess env — two independent bytecode-write sites broke the code signature seal on launch), verified via a real package:mac:dir build (codesign --verify --deep --strict clean before and after a full engine+guardian launch cycle), then the user ran Task 4's real-hardware Gatekeeper checkpoint on Apple Silicon: PASS. 01-03-SUMMARY.md written, status complete."
state_head: ff1662d55fb1aff30c9b0694e03f8ab578c54267
progress:
  total_phases: 6
  completed_phases: 0
  total_plans: 13
  completed_plans: 5
  percent: 0
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-09-17)

**Core value:** The app runs and works correctly on macOS: build, launch, spawn agents, and manage their work end-to-end — matching what the Windows build already does.
**Current focus:** Phase 1 complete (PKG-01, PKG-02, PKG-03, RUN-01 all met, Task 4 human checkpoint PASS on 2026-09-23). Ready to plan/execute Phase 2 (process lifecycle).

## Current Position

Phase: 1 of 5 (Packaging & Runtime Foundation)
Plan: 3 of 3 in current phase — Phase 1 complete (12 plans total across phases 1-5)
Status: Phases 1 and 02.1 executed and complete; phases 2, 3, 4, 5 planned, not started
Last activity: 2026-09-23 — PKG-03's code fix (PYTHONDONTWRITEBYTECODE=1 in both the engine spawn and its guardian subprocess) is committed and verified: codesign --verify --deep --strict passes both before and after a real launch/shutdown cycle. Task 4 (blocking-human checkpoint) then run by the user on real Apple Silicon: PASS — in-place launch dialog-free bar one unrelated TCC privacy prompt, seal intact post-quit, quarantined copy showed the correct recoverable Gatekeeper dialog. Phase 1 closed: PKG-01, PKG-02, PKG-03, RUN-01 all met (see 01-03-SUMMARY.md). Intel/x64 verification still a documented follow-up (no Intel hardware this session).

Progress: [░░░░░░░░░░] 0%

## Performance Metrics

Total plans completed: 0
Average duration per plan: N/A
Recent trend: N/A (no plans executed yet)
**Per-Plan Metrics:**

| Plan | Duration | Tasks | Files |
|------|----------|-------|-------|
| Phase 02.1 P01 | 50min | 3 tasks | 4 files |
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

**Stopped at:** Completed 02-03-PLAN.md

Last session: 2026-09-23T09:51:39.721Z
Last completed: Phase 1 closed — PKG-03 code fix (both spawn sites), verification-trap doc/plan cleanup, and Task 4's real-hardware human checkpoint (PASS), all this session. Phase 02.1 execution and Phase 1 Tasks 1-3 completed 2026-09-17.
Resume file: None

## Decisions

- [Phase 1]: PosixTree adapter mirrors WindowsTree's exact public surface; kqueue EVFILT_PROC/NOTE_EXIT for exit detection, ps-based enumerate-then-killpg sweep for teardown
- [Phase 1]: Escaped-process-group descendants are only provably swept when the guardian's teardown runs while the engine is still alive (parent-death/guardian-crash); the engine-dies-first case has a documented residual gap (see 02.1-01-SUMMARY.md Threat Flags)
- [Phase 1]: 02-03: verification-only — no RED/GREEN split; a single test(...) commit both wrote and green-lit the precedence-order proof since no production code changes were permitted
