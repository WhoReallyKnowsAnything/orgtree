# Roadmap: Orgtree (macOS fork)

## Overview

The macOS port shipped under GSD (phases 1-5, archived in `.planning-gsd/`). This milestone (v2.2.0) first closes the port's loose ends so there is a green macOS baseline, then pulls in the 369 upstream commits against that baseline, then turns the unsigned local build into real distribution. Distribution comes last because signing needs an Apple Developer account and auto-update depends on signing.

## Phases

- [ ] **Phase 1: macOS Hardening** - Green test suite on macOS and real-hardware proof of launchd autostart
- [ ] **Phase 2: Upstream Sync** - Merge current `upstream/main` with macOS support and the Windows build both intact
- [ ] **Phase 3: Signed Distribution** - Signed, notarized `.dmg` that opens without Gatekeeper overrides
- [ ] **Phase 4: macOS Auto-Update** - In-app updater installs a newer signed release on macOS
- [ ] **Phase 5: Autostart Toggle** - Settings control that turns LaunchAgent autostart on and off

## Phase Details

### Phase 1: macOS Hardening
**Goal:** A clean macOS baseline: every test passes and autostart is proven on real hardware.
**Depends on:** Nothing (first phase)
**Requirements:** HARD-01, HARD-02, HARD-03
**Success Criteria:**
1. `npm test` on macOS exits 0, with `tests/update-rehearsal.test.mjs` included and none of its cases skipped
2. After a full reboot with the app never opened, `launchctl print gui/$(id -u)/<Label>` shows the engine running and its HTTP endpoint answers
3. With the engine forced into a crash loop, the app shows the remediation dialog and does not report "disabled"
4. A phase VERIFICATION report records the reboot and crash-loop results

### Phase 2: Upstream Sync
**Goal:** The fork contains current `upstream/main` with no regression on either platform.
**Depends on:** Phase 1
**Requirements:** SYNC-01, SYNC-02, SYNC-03
**Success Criteria:**
1. `git rev-list --count HEAD..upstream/main` returns 0
2. The macOS `.app` builds, launches, and completes a real agent turn through the acceptance runner
3. `npm test` on macOS exits 0 after the merge
4. The Windows NSIS installer builds and installs on a Windows machine or CI runner

### Phase 3: Signed Distribution
**Goal:** Users get a `.dmg` that macOS trusts.
**Depends on:** Phase 2
**Requirements:** DIST-01
**Success Criteria:**
1. `codesign --verify --deep --strict` passes on the packaged `.app`
2. `spctl --assess --type execute` accepts the app and `xcrun stapler validate` passes on the `.dmg`
3. On a Mac that has never run Orgtree, the downloaded `.dmg` opens and launches without right-click Open or `xattr` workarounds

### Phase 4: macOS Auto-Update
**Goal:** macOS users receive updates in-app, as Windows users already do.
**Depends on:** Phase 3
**Requirements:** DIST-02
**Success Criteria:**
1. An installed older signed build detects a newer published release and shows the update notice
2. Accepting the update installs it and the app relaunches reporting the new version
3. The updated app still passes `codesign --verify --deep --strict`

### Phase 5: Autostart Toggle
**Goal:** Users control engine autostart without touching `launchctl`.
**Depends on:** Phase 1
**Requirements:** DIST-03
**Success Criteria:**
1. Turning the toggle off removes the LaunchAgent: `launchctl print gui/$(id -u)/<Label>` fails and no plist remains in `~/Library/LaunchAgents/`
2. Turning it on reinstalls it, and the engine is running after a reboot
3. The toggle's displayed state matches the LaunchAgent's real state after an app restart
