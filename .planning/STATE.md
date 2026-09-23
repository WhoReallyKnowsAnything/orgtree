---
gsd_state_version: "1.0"
current_phase: 1
current_phase_name: Packaging & Runtime Foundation
status: planning
stopped_at: 01-03-PLAN.md Task 4 (blocking-human checkpoint)
last_updated: "2026-09-23T09:16:10.000Z"
last_activity: 2026-09-23
last_activity_desc: "Resumed Phase 1, fixed PKG-03: PYTHONDONTWRITEBYTECODE=1 added to both the packaged engine spawn (apps/desktop/main/engine.ts) and its guardian subprocess env (engine/process_lifetime.py) — two independent bytecode-write sites broke the code signature seal on launch. Verified via real package:mac:dir build: codesign --verify --deep --strict passes before AND after a full engine+guardian launch/shutdown cycle. 01-03-SUMMARY.md written. Still blocked on Task 4's real-hardware Gatekeeper dialog check."
state_head: 4af4984ea27c3e90cbb145bda5d5fff13eef9583
progress:
  total_phases: 6
  completed_phases: 0
  total_plans: 13
  completed_plans: 1
  percent: 0
---

# Project State

## Project Reference

See: .planning/PROJECT.md (updated 2026-09-17)

**Core value:** The app runs and works correctly on macOS: build, launch, spawn agents, and manage their work end-to-end — matching what the Windows build already does.
**Current focus:** Phase 1 partially executed — PKG-03 code fix complete and verified; blocked on Task 4's real-hardware Gatekeeper checkpoint

## Current Position

Phase: 1 of 5 (Packaging & Runtime Foundation)
Plan: 1 of 3 in current phase (12 plans total across phases 1-5)
Status: Phases 1 and 02.1 executed (Phase 1 INCOMPLETE — Task 4 human checkpoint pending); phases 2, 3, 4, 5 planned, not started
Last activity: 2026-09-23 — PKG-03's code fix (PYTHONDONTWRITEBYTECODE=1 in both the engine spawn and its guardian subprocess) is committed and verified: codesign --verify --deep --strict passes both before and after a real launch/shutdown cycle. RUN-01 and PKG-02 confirmed unregressed. PKG-01 now fully met (bundle passes strict verify, not just -dv). PKG-03 code-level cause resolved; still blocked on Task 4's real-hardware Gatekeeper dialog check (see 01-03-SUMMARY.md).

Progress: [░░░░░░░░░░] 0%

## Performance Metrics

Total plans completed: 0
Average duration per plan: N/A
Recent trend: N/A (no plans executed yet)
**Per-Plan Metrics:**

| Plan | Duration | Tasks | Files |
|------|----------|-------|-------|
| Phase 02.1 P01 | 50min | 3 tasks | 4 files |

## Accumulated Context

**Decisions:** See .planning/PROJECT.md Key Decisions. Roadmap-level decision: sequence packaging (Phase 1) before process lifecycle (Phase 2) before autostart (Phase 3), since nothing is testable on real hardware until a launchable `.app` exists, and the LaunchAgent (Phase 3) invokes the engine ported in Phase 2. UI/OS parity (Phase 4) is isolated and deferred until after the harder plumbing; verification (Phase 5) closes the milestone.

**Pending Todos:** None captured yet.

**Blockers/Concerns:**

- Phase 1: AMFI enforcement on Apple Silicon blocks unsigned binaries even offline ("app is damaged" dialog) — mitigated by ad-hoc signing (`mac.identity: "-"`), but must be verified on real Apple Silicon hardware early (PKG-03).
- Phase 1: nested unsigned binaries inside the `.app` bundle can re-trigger the AMFI failure if ad-hoc signing doesn't cover the full bundle, not just the top-level `.app`.
- Phase 3: `openAtLogin`/LaunchAgent reliability under an unsigned build is a documented caveat, not yet a tested outcome — verify by hand.
- ~~Owned by Phase 2.1 (PROC-04): `arm_process_lifetime` raises unconditionally when `os.name != "nt"`~~ — RESOLVED 2026-09-17 by 02.1-01-PLAN.md (commits `30dec80`/`fbc9fc4`/`4af4984` on `session/execute-phase-2-1-engine-process-lifetim`): `PosixTree` adapter implemented, `engine/launch.py::main()` self-setpgids, `LifetimeTests`/`LaunchRefusalTests` un-skipped and passing on macOS. Residual gap (NOT resolved): a descendant in its own escaped process group is only provably swept when the guardian's teardown runs while the engine is still alive — see `02.1-01-SUMMARY.md` Threat Flags and `.planning/WINDOWS.md` entry #1.
- **RESOLVED 2026-09-23 (PKG-03 code-level cause) — STILL BLOCKS Phase 1 closure on Task 4's human checkpoint.** Cause was two independent code paths writing `__pycache__/*.cpython-313.pyc` into the signed `Contents/Resources/engine/` tree at launch: the main engine spawn (`apps/desktop/main/engine.ts`) and its guardian watchdog re-exec (`engine/process_lifetime.py::arm_process_lifetime`, which builds a deliberately strict allowlisted env for the child and silently dropped `PYTHONDONTWRITEBYTECODE` even after the first fix). Fix: `PYTHONDONTWRITEBYTECODE=1` set explicitly in both spawn sites (commits `6175497`, `d9021bd`). Verified on a real `package:mac:dir` build: `codesign --verify --deep --strict` passes both before and after a full launch/shutdown cycle, 0 `__pycache__` anywhere in the bundle. Also replaced the `codesign -dv` verification-trap checks in the plan and `docs/macos-first-launch.md` with `--verify --deep --strict` (commit `fd5f9fb`) — `-dv` only proves a signature exists, which is exactly how this bug shipped undetected the first time. **Remaining:** Task 4 (`01-03-PLAN.md`, `checkpoint:human-verify`, `gate="blocking-human"`) still needs a human on real Apple Silicon hardware to confirm the quarantined-copy dialog reads "developer cannot be verified" (not "is damaged") — `spctl`/`codesign` confirm Gatekeeper blocks the quarantined copy but can't distinguish the two dialog texts from the CLI. See `01-03-SUMMARY.md`.

## Session Continuity

**Stopped at:** `01-03-PLAN.md` Task 4 — `checkpoint:human-verify`, `gate="blocking-human"`. PKG-03's code fix is done and verified; only the real-hardware Gatekeeper dialog check is outstanding.

Last session: 2026-09-23T09:16:10.000Z
Last completed: PKG-03 code fix (both spawn sites) and verification-trap doc/plan cleanup, this session. Phase 02.1 execution and Phase 1 Tasks 1-3 completed 2026-09-17.
Resume file: `01-03-SUMMARY.md` (written this session, status: blocked). Resume by running Task 4 on real Apple Silicon hardware per its `how-to-verify` steps, then typing "approved" (or describing what failed) per its `resume-signal`.

## Decisions

- [Phase 1]: PosixTree adapter mirrors WindowsTree's exact public surface; kqueue EVFILT_PROC/NOTE_EXIT for exit detection, ps-based enumerate-then-killpg sweep for teardown
- [Phase 1]: Escaped-process-group descendants are only provably swept when the guardian's teardown runs while the engine is still alive (parent-death/guardian-crash); the engine-dies-first case has a documented residual gap (see 02.1-01-SUMMARY.md Threat Flags)
