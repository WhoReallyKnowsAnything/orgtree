# Roadmap: Orgtree (macOS Port)

## Overview

Orgtree currently only runs on Windows: NSIS packaging, a Scheduled Task for boot autostart, `taskkill`/`ctypes` process management, and Windows-only taskbar/menu behavior. This roadmap ports it to macOS in dependency order. Nothing is testable on real hardware until a launchable `.app` exists, so packaging comes first. Process lifecycle is ported next because the autostart mechanism (Phase 3) invokes the engine and needs correct POSIX process management underneath it. UI/OS-integration parity is the most isolated work and is sequenced after the harder plumbing. The port closes with an end-to-end verification phase that proves a real agent job runs start-to-finish on macOS and that the Windows-only test skips have macOS counterparts.

## Phases

**Phase Numbering:**

- Integer phases (1, 2, 3): Planned milestone work
- Decimal phases (2.1, 2.2): Urgent insertions (marked with INSERTED)

Decimal phases appear between their surrounding integers in numeric order.

- [ ] **Phase 1: Packaging & Runtime Foundation** - Produce a launchable, ad-hoc-signed macOS `.app` with a bundled Python runtime and `.icns` icon
- [ ] **Phase 2: Process Lifecycle Port** - Port provider CLI resolution, liveness checks, and process-tree termination to POSIX
- [ ] **Phase 3: Launchd Autostart** - Replace the Windows Scheduled Task with a `launchd` LaunchAgent, with crash-loop detection
- [ ] **Phase 4: macOS UI & OS Integration Parity** - Dock/menu/tray behavior and update notices match native macOS conventions
- [ ] **Phase 5: End-to-End Verification & Test Coverage** - Prove a real agent job runs on macOS and close the Windows-only test gaps

## Phase Details

### Phase 1: Packaging & Runtime Foundation

**Goal**: An unsigned Orgtree `.app` builds and launches on real Apple Silicon and Intel Macs, with its Python engine runtime bundled inside it.
**Depends on**: Nothing (first phase)
**Requirements**: PKG-01, PKG-02, PKG-03, RUN-01
**Success Criteria** (what must be TRUE):

  1. Running the mac build produces a `.app` that launches on Apple Silicon without an AMFI "app is damaged" failure, because it is ad-hoc signed (`mac.identity: "-"`).
  2. The app's Dock and Finder icon displays as the real Orgtree icon, not a blank or default icon, because a `.icns` (generated via `iconutil`) is bundled instead of the Windows-only `.ico`.
  3. A first-time user who hits the Gatekeeper block on double-click can follow documented steps (right-click → Open, or System Settings approval) to launch the app successfully — verified by hand on real Apple Silicon hardware.
  4. `tools/provision-runtime.py` downloads and stages an arch-correct (`aarch64`/`x86_64`) `python-build-standalone` runtime into the build instead of hard-exiting on non-`win32`.
**Plans**: 3 plans
Plans:
- [ ] 01-01-PLAN.md — macOS runtime provisioning and layout verification (RUN-01)
- [ ] 01-02-PLAN.md — .icns icon generation (PKG-02)
- [ ] 01-03-PLAN.md — electron-builder mac target, ad-hoc signing, Gatekeeper docs (PKG-01, PKG-03)

### Phase 2: Process Lifecycle Port

**Goal**: The Python engine correctly finds, runs, monitors, and terminates provider CLI subprocesses on macOS.
**Depends on**: Phase 1
**Requirements**: PROC-01, PROC-02, PROC-03
**Success Criteria** (what must be TRUE):

  1. The engine locates and launches Claude Code, Codex, and Antigravity CLIs from their real macOS install paths (npm shims, Antigravity's own install location) via `shutil.which()`.
  2. The engine's liveness check correctly reports whether a provider process is alive or dead on macOS, matching real process state (`os.kill(pid, 0)` verified against actual macOS behavior).
  3. Stopping an agent kills the provider CLI and all of its descendant processes on macOS — including `codexrun.py`'s process, once it spawns with `start_new_session` like `gitrunner.py` already does — leaving no orphaned processes.
**Plans**: 3 plans

Plans:
- [ ] 02-01-PLAN.md — codexrun.py + antigravityrun.py: start_new_session spawn + os.killpg termination (PROC-03)
- [ ] 02-02-PLAN.md — supervisor.py (4 spawn sites + _wd_kill_tree) + mailhub_runtime.py orphan reclaim (PROC-03, D-07 scope)
- [ ] 02-03-PLAN.md — PROC-01/PROC-02 verification: mocked-path resolver tests + liveness confirmation

### Phase 3: Launchd Autostart

**Goal**: The engine starts automatically at login on macOS, and the user is alerted if that autostart silently stops working.
**Depends on**: Phase 1, Phase 2
**Requirements**: BOOT-01, BOOT-02
**Success Criteria** (what must be TRUE):

  1. After installing the per-user LaunchAgent plist (`~/Library/LaunchAgents/`) and logging in, the Python engine is already running without the user launching it manually — no Windows Scheduled Task or registry mechanism involved.
  2. The LaunchAgent plist validates with `plutil -lint` and installs/uninstalls cleanly via `launchctl bootstrap`/`bootout`.
  3. If macOS silently disables the LaunchAgent after a crash loop, the app detects this and prompts the user instead of failing to start with no explanation.

**Plans**: 2 plans
Plans:
**Wave 1**

- [ ] 03-01-PLAN.md — Build launchagent-mac.ts: plist gen/lint/bootstrap/bootout (tracer, real round-trip), detectState() classification, remediation dialog content

**Wave 2** *(blocked on Wave 1 completion)*

- [ ] 03-02-PLAN.md — Wire darwin autostart into app startup; reboot-survival and extended crash-loop human-check items

### Phase 4: macOS UI & OS Integration Parity

**Goal**: Orgtree looks and behaves like a native macOS app instead of a ported Windows app.
**Depends on**: Phase 1
**Requirements**: UI-01, UI-02, UI-03, UI-04, UI-05, UPD-01
**Success Criteria** (what must be TRUE):

  1. When an org needs attention, the Dock icon bounces (`app.dock.bounce('critical')`) instead of relying on the Windows-only `flashFrame` window flash.
  2. Closing the last Orgtree window leaves the app running in the Dock; clicking the Dock icon reopens a window (`window-all-closed`/`activate` follow macOS convention, not Windows close-quits-app behavior).
  3. Orgtree has a native macOS app menu in the menu bar, where none exists today.
  4. The Dock badge count and the menu-bar tray icon both reflect live app state — pending tickets/mail via `app.setBadgeCount()`, and light/dark appearance via a Template tray image.
  5. The user sees a "new version available" notice when a newer release exists, without the app attempting to auto-apply the update.
**Plans:** 2 plans
Plans:
- [ ] 04-01-PLAN.md — Mac update notice end-to-end (bridge platform flag, openReleasePage IPC, tray mirror) + Dock bounce/badge sharing one identity set (UPD-01, UI-01, UI-04)
- [ ] 04-02-PLAN.md — Window lifecycle recreate-on-show, native App menu, tray Template icon (UI-02, UI-03, UI-05)
**UI hint**: yes

### Phase 5: End-to-End Verification & Test Coverage

**Goal**: The macOS port is proven to work end-to-end and has automated coverage in place of the Windows-only skips.
**Depends on**: Phase 1, Phase 2, Phase 3, Phase 4
**Requirements**: VER-01, VER-02
**Success Criteria** (what must be TRUE):

  1. A user can package, launch, and run a full agent job on macOS — spawn an agent, watch it do real work, and see the result land back in Orgtree — with no manual workarounds.
  2. Test runs on macOS exercise the scenarios that were previously Windows-only skips (installer, elevation-equivalent, taskbar/dock probes, `test_service_host.py`) instead of silently skipping them.
**Plans**: 2 plans
Plans:
- [ ] 05-01-PLAN.md — Real end-to-end agent-turn acceptance runner + process-lifecycle pre-flight gate (VER-01)
- [ ] 05-02-PLAN.md — POSIX descriptor-protection fix + macOS test-skip coverage closure (VER-02)

## Progress

| Phase | Plans Complete | Status | Completed |
|-------|----------------|--------|-----------|
| 1. Packaging & Runtime Foundation | 0/TBD | Not started | - |
| 2. Process Lifecycle Port | 0/TBD | Not started | - |
| 3. Launchd Autostart | 0/2 | Not started | - |
| 4. macOS UI & OS Integration Parity | 0/TBD | Not started | - |
| 5. End-to-End Verification & Test Coverage | 0/TBD | Not started | - |
