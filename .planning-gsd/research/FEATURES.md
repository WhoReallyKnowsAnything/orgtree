# Feature Research

**Domain:** macOS-native desktop app parity for an Electron app (OS integration: dock/tray, attention/badging, auto-update, boot-time background process)
**Researched:** 2026-09-17
**Confidence:** MEDIUM (Electron/electron-builder official docs via Context7 = MEDIUM tier; Squirrel.Mac signing requirement cross-checked across electron/electron#8204, electron/electron#36640, electron-builder#3983, and Electron's own code-signing docs = MEDIUM verified tier. No LOW-confidence claims are presented as fact below.)

## Context

Orgtree's Windows build uses `window.setAppDetails()` (taskbar AppUserModelId grouping), `flashFrame` via a custom `taskbar-attention.ts` (cross-org "needs attention" flashing), electron-updater's NSIS/Squirrel.Windows auto-update path, and a Windows Scheduled Task (`tools/*-boot-engine-task.ps1`) for boot-time engine autostart. All four are Windows-only APIs/mechanisms with no direct 1:1 macOS equivalent. This document maps each to its macOS counterpart and scopes what's genuinely required vs skippable for an **unsigned, local, personal macOS build** (per PROJECT.md MAC-01: unsigned, local build; no Apple Developer Program membership assumed).

## Feature Landscape

### Table Stakes (Users Expect These)

Features users assume exist on any native-feeling macOS app. Missing these makes the port feel like a Windows app running under a compatibility shim.

| Feature | Why Expected | Complexity | Notes |
|---|---|---|---|
| Dock icon shows correctly (`.icns`, not stretched `.ico`) | Every macOS app has a proper dock icon; a missing/wrong icon is the first thing a user notices | LOW | Generate `.icns` from existing `orgtree-eye` artwork (`iconutil` or `electron-icon-builder`). Already flagged as MAC-02/blocker #1 in CONCERNS.md. |
| App survives "all windows closed" without quitting | macOS convention: closing the last window does NOT quit the app (unlike Windows). App stays live in the Dock/menu bar until Cmd+Q or Dock "Quit" | LOW–MED | Must add a macOS-specific `window-all-closed` handler that does *not* call `app.quit()`, plus an `activate` handler that re-creates/re-shows the window. Getting this wrong makes the engine/tray silently die when the user closes the window, breaking the "background agent orchestration" value prop entirely. |
| Menu bar / Tray icon renders as a proper template image | `Tray` API already cross-platform (existing `traylist.ts` code should mostly just work), but macOS expects monochrome "template" icons that auto-adapt to light/dark menu bar, not a colored Windows-style icon | LOW | Ship a `*Template.png` (or `.pdf`) tray icon variant; Electron's `nativeImage` auto-detects the `Template` suffix. Cosmetic but a strong "looks native" signal. |
| Standard macOS app menu (About/Preferences/Services/Hide/Quit) | Every macOS app has the leftmost app-name menu with Cmd+Q, Cmd+H, Cmd+, conventions; without it the app feels foreign and some system shortcuts (Cmd+Q, Cmd+H) won't work at all | LOW | Build a minimal `Menu.buildFromTemplate` app menu for `darwin` alongside the existing Windows menu code path. |
| Dock "needs attention" signal (replaces `flashFrame`) | Windows taskbar flash has a real macOS analog users expect from any app that needs to interrupt them (Slack, Mail, etc. all bounce/badge the dock icon) | LOW | `app.dock.bounce('critical')` (bounces until app is focused or cancelled) is the direct swap-in for the existing `taskbar-attention.ts` flash logic. Works fine unsigned/locally — no signing dependency. |
| Boot-time engine autostart (replaces Scheduled Task) | This is an explicit Windows-parity requirement (CONCERNS.md #5, PROJECT.md core value: "matching what the Windows build already does") — without it, the headless engine doesn't survive reboots/logout the way it does on Windows | MED–HIGH | Real macOS equivalent is a hand-written `launchd` **LaunchAgent** plist (`~/Library/LaunchAgents/com.orgtree.engine.plist`, loaded via `launchctl bootstrap gui/$UID <plist>` or legacy `launchctl load`). This is a genuine rewrite of all 4 `.ps1` scripts (`register/unregister/probe/boot-engine-task`), not a port. **Important for unsigned builds: a raw LaunchAgent plist does NOT require code signing or notarization** — it's a classic launchd job, unrelated to the modern `SMAppService`/System-Settings-Login-Items path that Electron's own `setLoginItemSettings` now uses (see below). This keeps boot autostart fully achievable for a personal unsigned build. |
| Unsigned-app first-launch friction is handled/documented | Any unsigned/ad-hoc-signed local build triggers Gatekeeper ("app can't be opened because it is from an unidentified developer" / "app is damaged") on first launch | LOW | Not a code fix — a one-time user action: right-click → Open (bypasses Gatekeeper once), or approve via System Settings → Privacy & Security → "Open Anyway" after the first blocked attempt. On Apple Silicon, per electron-builder's own guidance, unsigned/ad-hoc apps can be approved for local use this way. Document this in the release README rather than trying to engineer around it. |

### Differentiators (Nice Native Polish — Not Required)

Features that make the macOS build feel *more* native than the bare minimum, but whose absence doesn't break anything users depend on.

| Feature | Value Proposition | Complexity | Notes |
|---|---|---|---|
| Dock badge count ("N orgs need attention") | More persistent/informative than a transient bounce — matches Slack/Mail-style dock badges | LOW | `app.setBadgeCount(n)` is the modern, simplest cross-platform (macOS+Linux) API — no notification-permission wrinkles reported for it, unlike the older `dock.setBadge(text)` API (Electron's own doc for that one notes a notification-permission dependency). Prefer `setBadgeCount` over `dock.setBadge` for this reason. Pairs naturally with `app.dock.bounce()` for the transient case. |
| `nativeTheme`-aware light/dark UI | macOS users expect apps to follow system appearance | LOW–MED | Only matters if the renderer doesn't already theme itself off `prefers-color-scheme`/`nativeTheme.shouldUseDarkColors`. Check existing renderer theming before treating this as new work. |
| Electron GUI "Launch Orgtree at login" toggle | Convenience parity with a typical macOS "open at login" preference | LOW (API) / MED (reliability) | `app.setLoginItemSettings({ openAtLogin: true })` is the API, but Electron's own docs carry an explicit caveat: **on macOS this goes through `SMAppService`, and "When an app isn't packaged, code signed, and notarized, `openAtLogin` may silently fail to take effect."** For an unsigned personal build, treat this as best-effort — test it on the actual packaged `.app`, and if it's flaky, fall back to telling the user to manually add the app in System Settings → General → Login Items. This is a *separate* concern from the engine's own LaunchAgent-based boot autostart above (that one is unaffected by this caveat). |
| Auto-update "check & notify" (not full auto-apply) | Some update signal is better than none for a personal build users forget to redownload | LOW–MED | See Anti-Features below — full Squirrel.Mac auto-update is off the table unsigned, but a lightweight "a newer version is available, click to open the download page" check (comparing a version string from a static JSON/GitHub release, no `autoUpdater` involved) is cheap and skippable-but-nice. |

### Anti-Features (Skip These — Would Add Real Cost for No Payoff on an Unsigned Personal Build)

| Feature | Why It Looks Appealing | Why It's a Trap Here | Do Instead |
|---|---|---|---|
| Full `electron-updater` auto-update on macOS (Squirrel.Mac / DMG+ZIP feed) | Windows build already has NSIS auto-update; seems like straightforward parity | **Hard blocker, not a config option:** Squirrel.Mac (the engine `electron-updater` uses on macOS) validates the code signature of the downloaded update against the running app and refuses to apply an unsigned update. Confirmed across electron/electron#8204, electron/electron#36640, electron-builder#3983, and Electron's code-signing docs — the failure mode is literally `Error: Could not get code signature for running application`. Code signing requires an active Apple Developer Program membership ($99/yr), explicitly out of scope for "unsigned, local build." | Ship version-check-only (compare a version string, link out to manual re-download) or skip auto-update on macOS entirely for now; document it as a known gap, not a bug. |
| Replicating Windows taskbar AppUserModelId grouping (`setAppDetails` equivalent) | Feels like missing parity since the Windows code has a whole `taskbar.ts` module for it | Electron's own docs mark `setAppDetails` as Windows-only, no-op elsewhere — and there's nothing to port: macOS already groups all of an app's windows under one Dock icon by bundle identifier, which is the *entire purpose* `setAppDetails` serves on Windows. Building any macOS-side "taskbar grouping" module would be solving a problem macOS doesn't have. | Delete/no-op the macOS branch outright; don't build a parallel module. |
| Mac App Store (`mas`/`mas-dev`) target | electron-builder supports it out of the box, looks like "free" extra distribution | MAS sandboxing (mandatory entitlements, restricted child-process spawning) would directly conflict with this app's core architecture — spawning `Claude Code`/`Codex`/other provider CLIs as subprocesses is the entire product. Sandbox entitlements for arbitrary subprocess execution are not realistically obtainable for this use case. | Do not pursue MAS. Direct DMG/ZIP distribution only. |
| Code signing + notarization pipeline (`afterSign` + `@electron/notarize`, hardened runtime, entitlements) | CONCERNS.md #1 lists this as part of "the real fix"; feels incomplete without it | Requires a paid Apple Developer ID and an Apple ID/API key wired into CI — explicitly out of scope per PROJECT.md's "unsigned, local build" framing. Worth flagging as a *future* milestone if this ever needs to be distributed beyond the author, but not this one. | Use `mac.sign.identity: null` (skip signing) or `"-"` (ad-hoc, for local Apple Silicon runnability) in the `build.mac` electron-builder config. Document the Gatekeeper first-launch step instead. |
| `openAsHidden` login-item option | Existing Windows code may have a "start minimized" style flag | Electron 44 **removed** `openAsHidden` from `setLoginItemSettings` entirely (macOS 12 and below only, now dropped) — it's not just deprioritized, it doesn't exist in current Electron. | If a "start hidden in tray" behavior is wanted, implement it manually: check a launch flag/login-item state in app code and simply don't call `.show()` on the main window at startup. |

## Feature Dependencies

```
Boot-time engine autostart (LaunchAgent plist)
  └──independent of──> Electron GUI "launch at login" toggle (SMAppService)
       (these are two different mechanisms; do NOT conflate them —
        the engine's LaunchAgent works unsigned, the GUI's SMAppService-based
        toggle may not)

Dock bounce (`app.dock.bounce`) ──pairs well with──> Dock badge count (`app.setBadgeCount`)
  (bounce = transient interrupt, badge = persistent count; both replace `flashFrame`)

Full auto-update (Squirrel.Mac) ──requires──> Code signing + notarization
  (blocked for this milestone; do not attempt Squirrel.Mac without signing)

Version-check-only update notice ──has no dependency on──> code signing
  (safe fallback, works fully unsigned)
```

### Dependency Notes

- **Boot-time engine autostart is independent of the GUI login-item toggle:** these are commonly conflated because both are "start something automatically," but they use entirely different macOS mechanisms (raw `launchd` LaunchAgent plist vs. `SMAppService`-backed Login Items). The LaunchAgent path is unaffected by lack of code signing; the `SMAppService` path (used by Electron's `setLoginItemSettings`) explicitly is not guaranteed to work unsigned.
- **Full auto-update requires signing; nothing else does.** Every other macOS-parity feature in this document (dock bounce, badge, tray, boot autostart, app menu) works fine on an unsigned local build. Auto-update is the one genuine hard wall — plan around it rather than trying to work around it, since the blocker is Squirrel.Mac's own signature-verification logic, not a config flag.

## MVP Definition

### Launch With (v1 — required for Windows-parity claim)

- [ ] `.icns` dock icon — app looks broken without it (blocker #1 in CONCERNS.md)
- [ ] macOS-correct `window-all-closed`/`activate` lifecycle — without it the app quits when the user closes the window, breaking the always-on background-agent model
- [ ] Dock bounce (`app.dock.bounce('critical')`) replacing `flashFrame` — this is the direct, low-cost swap for the existing cross-org attention feature; skipping it is a visible regression from the Windows build
- [ ] `launchd` LaunchAgent plist replacing the Scheduled Task boot mechanism — explicitly called out as a required Windows-parity item in PROJECT.md/CONCERNS.md, and it's the one boot-autostart path that actually works unsigned
- [ ] Standard macOS app menu (minimum: About/Quit/Hide, Cmd+Q wired up) — without this, basic OS conventions don't work at all

### Add After Validation (v1.x)

- [ ] Dock badge count (`app.setBadgeCount`) — cheap addition once bounce/attention plumbing exists, adds a persistent counter that mirrors Slack/Mail conventions
- [ ] Tray icon as proper `Template` image — cosmetic polish, do once the base tray functionality is confirmed working
- [ ] `nativeTheme` dark-mode check — only if the renderer doesn't already handle it

### Future Consideration (v2+, or explicitly out of scope for a personal unsigned build)

- [ ] Code signing + notarization pipeline — needed only if this is ever distributed beyond the author; requires a paid Apple Developer account
- [ ] Full Squirrel.Mac auto-update — blocked until signing exists; revisit only alongside the signing pipeline
- [ ] Electron GUI "launch at login" toggle via `setLoginItemSettings` — best-effort only; the boot-autostart requirement is already satisfied by the engine's LaunchAgent, so this is a pure UX nicety, not a functional gap
- [ ] Mac App Store distribution — architecturally incompatible with this app's subprocess-spawning model; do not pursue

## Feature Prioritization Matrix

| Feature | User Value | Implementation Cost | Priority |
|---|---|---|---|
| `.icns` icon | HIGH | LOW | P1 |
| macOS window/quit lifecycle fix | HIGH | LOW-MED | P1 |
| Dock bounce (flashFrame replacement) | HIGH | LOW | P1 |
| `launchd` LaunchAgent boot autostart | HIGH | MED-HIGH | P1 |
| Native app menu | MED | LOW | P1 |
| Gatekeeper first-launch doc/handling | MED | LOW (docs only) | P1 |
| Dock badge count | MED | LOW | P2 |
| Tray Template icon polish | LOW | LOW | P2 |
| `setLoginItemSettings` GUI toggle | LOW | LOW (flaky payoff) | P3 |
| Version-check-only update notice | LOW-MED | LOW-MED | P3 |
| Code signing/notarization | — (out of scope this milestone) | HIGH (external account) | P3/deferred |
| Full Squirrel.Mac auto-update | — (blocked) | HIGH (blocked without signing) | Deferred |
| MAS distribution | — | — | Do not pursue |

**Priority key:**
- P1: Must have for the "matches Windows build" milestone goal
- P2: Should have, cheap polish once P1 lands
- P3: Nice to have, low payoff for an unsigned personal build; defer

## Windows API → macOS Equivalent Quick Reference

| Windows API/mechanism (existing code) | macOS equivalent | Signing dependency? |
|---|---|---|
| `window.setAppDetails()` (`taskbar.ts`) | None — no-op, not needed (Dock groups by bundle ID automatically) | N/A |
| `flashFrame` / custom `taskbar-attention.ts` | `app.dock.bounce('critical'\|'informational')` + `dock.cancelBounce(id)` | No |
| (implicit) attention badge | `app.setBadgeCount(n)` (preferred) or `dock.setBadge(text)` | No (setBadgeCount); `dock.setBadge` docs mention a notification-permission dependency |
| `electron-updater` + NSIS (Squirrel.Windows-style) | `electron-updater` + Squirrel.Mac (DMG+ZIP target) | **Yes — hard-blocked unsigned** |
| Windows Scheduled Task (S4U) boot autostart (`tools/*-boot-engine-task.ps1`) | `launchd` LaunchAgent plist (`~/Library/LaunchAgents/*.plist`, `launchctl bootstrap`) | No |
| (n/a — new ask) Electron "launch app at login" toggle | `app.setLoginItemSettings({ openAtLogin: true })` (`SMAppService`-backed on macOS 13+) | **Best-effort unsigned — Electron's own docs warn it "may silently fail" without packaging/signing/notarization** |

## Sources

- Electron official docs (`app.dock.bounce`, `dock.setBadge`, `app.setBadgeCount`, `app.getBadgeCount`, `app.setLoginItemSettings`, `SMAppService` mapping, Electron 44 breaking change removing `openAsHidden`) — via Context7 `/electron/electron`, MEDIUM confidence
- electron-builder official docs (`mac` target overview, auto-update auto-updatable targets, `sign.identity` / ad-hoc signing / Local Development vs CI table, glossary "ad-hoc signing") — via Context7 `/electron-userland/electron-builder`, MEDIUM confidence
- Squirrel.Mac unsigned-update failure, cross-checked across electron/electron#8204, electron/electron#36640, electron-builder#3983, and Electron's code-signing tutorial — WebSearch, MEDIUM confidence (verified/cross-checked across 3+ independent official-repo sources)
- Project codebase evidence: `.planning/codebase/ARCHITECTURE.md` (`taskbar.ts`, `taskbar-attention.ts`, `traylist.ts`, `updater.ts`/`installer-upgrade.ts` component table; "Windows-only taskbar API surface" and "Windows-only installer/update pipeline" pitfalls) and `.planning/codebase/CONCERNS.md` (blockers #1 packaging config, #2 NSIS installer, #4 auto-update flow, #5 boot-engine Scheduled Task → `launchd` note, #7 Windows-only native window chrome APIs) — HIGH confidence (direct codebase inspection)

---
*Feature research for: macOS Electron desktop app OS-integration parity (Orgtree macOS port)*
*Researched: 2026-09-17*
