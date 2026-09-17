# Requirements: Orgtree (macOS Port)

**Defined:** 2026-09-17
**Core Value:** The app runs and works correctly on macOS: build, launch, spawn agents, and manage their work end-to-end — matching what the Windows build already does.

## v1 Requirements

### Packaging & Build

- [ ] **PKG-01**: `electron-builder` config gains a working `mac` target producing an unsigned/ad-hoc-signed local `.app` (`mac.identity: null` or `"-"`)
- [ ] **PKG-02**: App ships `.icns` icon assets (generated via `iconutil`), replacing the `.ico`-only assets
- [ ] **PKG-03**: First-launch Gatekeeper/AMFI approval flow is documented for the user (right-click→Open / System Settings approval) — no code fix exists, this is doc + a manual verification step on real Apple Silicon hardware early in the port

### Python Runtime

- [ ] **RUN-01**: `tools/provision-runtime.py` provisions a macOS Python runtime via `python-build-standalone` (arch-specific `aarch64`/`x86_64` `install_only` tarball), replacing its current hard-exit on non-`win32`

### Process Lifecycle (POSIX)

- [ ] **PROC-01**: Executable resolution (`providers.py` and related) resolves provider CLIs (Claude Code, Codex, Antigravity) correctly on macOS — verify existing `shutil.which()` fallback covers real macOS install paths (npm shims, Antigravity install location)
- [ ] **PROC-02**: Process liveness checks work on macOS — verify existing POSIX branch in `liveness.py` (`os.kill(pid, 0)`) against real macOS behavior
- [ ] **PROC-03**: Child-process containment (killing a provider CLI kills its descendants, not just the parent) works on macOS for every process spawn path, including `codexrun.py::CodexProcess.close()` where `start_new_session` is currently missing — matching the pattern already correct in `gitrunner.py`

### Boot Autostart

- [ ] **BOOT-01**: Engine autostart on login is implemented via a per-user `launchd` LaunchAgent plist (`~/Library/LaunchAgents/`), replacing the Windows Scheduled Task + registry mechanism
- [ ] **BOOT-02**: App detects and surfaces to the user when macOS has silently disabled the LaunchAgent after a crash loop (no programmatic re-enable exists — must prompt, not fail silently)

### macOS UI Parity

- [ ] **UI-01**: Cross-org "attention" signal (currently Windows `flashFrame`) uses `app.dock.bounce('critical')` on macOS
- [ ] **UI-02**: Window lifecycle follows macOS convention (`window-all-closed` doesn't quit the app; `activate` reopens the window) — currently only Windows-style close-quits-app behavior exists
- [ ] **UI-03**: App has a native macOS app menu (currently none — Windows relies on the OS-provided menu bar behavior)
- [ ] **UI-04**: Dock badge count reflects pending tickets/mail via `app.setBadgeCount()`
- [ ] **UI-05**: Tray icon uses a Template image so it adapts to light/dark macOS menu bar

### Update Notice

- [ ] **UPD-01**: App shows a "new version available" notice (version-check only, no auto-apply) — full auto-update via `electron-updater`/Squirrel.Mac is blocked by code-signing requirements this project doesn't meet

### Verification

- [ ] **VER-01**: Packaged unsigned `.app` builds, launches, and completes a real end-to-end agent job on macOS (spawn agent → agent does work → result lands back in orgtree)
- [ ] **VER-02**: Windows-only test suite (installer/elevation/taskbar probes, `test_service_host.py` Windows-only skips) gets macOS counterparts so mac-specific logic has real coverage instead of silent skips

## v2 Requirements

Deferred — acknowledged but not in current roadmap.

### Distribution

- **DIST-01**: Code-signed, notarized `.dmg` distribution (requires Apple Developer account)
- **DIST-02**: Full `electron-updater` auto-update on macOS (depends on DIST-01)
- **DIST-03**: `setLoginItemSettings`-based GUI toggle for autostart, layered on top of the `launchd` LaunchAgent from BOOT-01

## Out of Scope

| Feature | Reason |
|---------|--------|
| Signed/notarized `.dmg` distribution | No Apple Developer account; unsigned local build is sufficient for v1 (deferred to DIST-01) |
| Mac App Store distribution | MAS sandboxing conflicts with this app's core subprocess-spawning architecture — never planned |
| NSIS-equivalent `.pkg` installer | Unsigned `.app` via `electron-builder --mac --dir` is enough for v1 |
| Linux support | Not requested; no Linux-specific findings surfaced during codebase mapping or research |
| Full auto-update on macOS | Blocked by code signing (Squirrel.Mac refuses unsigned updates); version-check notice (UPD-01) covers v1 needs |

## Traceability

Filled in during roadmap creation.

| Requirement | Phase | Status |
|-------------|-------|--------|
| PKG-01 | TBD | Pending |
| PKG-02 | TBD | Pending |
| PKG-03 | TBD | Pending |
| RUN-01 | TBD | Pending |
| PROC-01 | TBD | Pending |
| PROC-02 | TBD | Pending |
| PROC-03 | TBD | Pending |
| BOOT-01 | TBD | Pending |
| BOOT-02 | TBD | Pending |
| UI-01 | TBD | Pending |
| UI-02 | TBD | Pending |
| UI-03 | TBD | Pending |
| UI-04 | TBD | Pending |
| UI-05 | TBD | Pending |
| UPD-01 | TBD | Pending |
| VER-01 | TBD | Pending |
| VER-02 | TBD | Pending |

**Coverage:**
- v1 requirements: 17 total
- Mapped to phases: 0
- Unmapped: 17 ⚠️ (resolved by roadmap creation)

---
*Requirements defined: 2026-09-17*
*Last updated: 2026-09-17 after initial definition*
