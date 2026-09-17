---
gsd_state_version: "1.0"
current_phase: 1
current_phase_name: Packaging & Runtime Foundation
status: planning
stopped_at: Completed 02.1-01-PLAN.md
last_updated: "2026-09-17T18:59:05.805Z"
last_activity: 2026-09-17
last_activity_desc: "phases 1-5 planned in parallel worktrees and merged to main. Plan counts: P1 3, P2 3, P3 2, P4 2, P5 2. All plan-checker verdicts pass. Requirement coverage confirmed across PKG/RUN, PROC, BOOT, UI/UPD, VER."
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
**Current focus:** Phase 1 partially executed — PKG-03 blocked, paused 2026-09-17

## Current Position

Phase: 1 of 5 (Packaging & Runtime Foundation)
Plan: 1 of 3 in current phase (12 plans total across phases 1-5)
Status: Phases 1 and 02.1 executed (Phase 1 INCOMPLETE — PKG-03 fails); phases 2, 3, 4, 5 planned, not started
Last activity: 2026-09-17 — Phase 1 executed except PKG-03. RUN-01 met (macOS runtime provisioned, staged, and resolved at launch; engine verified starting on Apple Silicon with a live child tree). PKG-02 met (orgtree-eye.icns renders, confirmed visually). PKG-01 partial: the .app and nested bin/python3.13 are ad-hoc signed, but the bundle fails codesign --verify --strict. PKG-03 NOT met — see Blockers. Paused here at WhoReallyKnowsAnything's request.

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
- **OPEN (PKG-03, blocks Phase 1 closure): the packaged `.app` fails `codesign --verify --deep --strict` with "a sealed resource is missing or invalid".** Cause: running the app writes `__pycache__/*.cpython-313.pyc` files into `Contents/Resources/engine/` AFTER signing, so the bundle no longer matches its seal. Confirmed 2026-09-17 on real Apple Silicon: a quarantined copy produces the unrecoverable "Orgtree.app is damaged and can't be opened" dialog (Cancel / Move to Bin only), NOT the recoverable "developer cannot be verified" Gatekeeper dialog that `docs/macos-first-launch.md` documents. The original build in the worktree fails verification identically, so this is not copy corruption. This would break the first launch of any installed copy, not just this test. Fix needs two parts: (1) keep bytecode out of the bundle — `PYTHONDONTWRITEBYTECODE=1`/`-B` for the packaged engine, or pre-compile and seal `.pyc` at build time, or exclude `__pycache__` in electron-builder's file rules; (2) ordering — any verification that RUNS the app must happen before signing, or be followed by a re-sign. NOTE the verification trap that hid this: `codesign -dv` only shows a signature EXISTS; only `codesign --verify --strict` proves it is VALID. Phase 1's checks used `-dv`.

## Session Continuity

**Stopped at:** Phase 1 paused mid-execution 2026-09-17 — PKG-03 unresolved (see Blockers). WhoReallyKnowsAnything to pick up later.

Last session: 2026-09-17T18:59:05.793Z
Last completed: Phase 02.1 execution, and Phase 1 execution except PKG-03.
Resume file: None — resume by fixing the PKG-03 `__pycache__` seal-invalidation blocker above, then re-running the Gatekeeper quarantine check. `01-03-SUMMARY.md` was deliberately never written, so Phase 1 is not closed.

## Decisions

- [Phase 1]: PosixTree adapter mirrors WindowsTree's exact public surface; kqueue EVFILT_PROC/NOTE_EXIT for exit detection, ps-based enumerate-then-killpg sweep for teardown
- [Phase 1]: Escaped-process-group descendants are only provably swept when the guardian's teardown runs while the engine is still alive (parent-death/guardian-crash); the engine-dies-first case has a documented residual gap (see 02.1-01-SUMMARY.md Threat Flags)
