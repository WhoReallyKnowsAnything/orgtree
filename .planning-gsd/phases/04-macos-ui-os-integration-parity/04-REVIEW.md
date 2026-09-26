---
phase: 04-macos-ui-os-integration-parity
reviewed: 2026-09-23T00:00:00Z
depth: standard
files_reviewed: 15
files_reviewed_list:
  - apps/desktop/main/index.ts
  - apps/desktop/main/taskbar-attention.ts
  - apps/desktop/main/updater.ts
  - apps/desktop/preload/index.ts
  - apps/desktop/renderer/src/App.tsx
  - apps/desktop/renderer/src/update-notice.tsx
  - packages/contracts/index.ts
  - tests/app-menu-wiring.test.mjs
  - tests/electron.probe.ts
  - tests/icon-assets.test.mjs
  - tests/taskbarattention.test.mjs
  - tests/tray-icon-wiring.test.mjs
  - tests/update-notice-ui.test.mjs
  - tests/updater.test.mjs
  - tests/window-recreate.test.mjs
findings:
  critical: 1
  warning: 2
  info: 1
  total: 4
status: issues_found
---

# Phase 04: Code Review Report

**Reviewed:** 2026-09-23T00:00:00Z
**Depth:** standard
**Files Reviewed:** 15
**Status:** issues_found

## Summary

Reviewed the macOS UI/OS integration parity work: the manual-upgrade update notice (mac never auto-installs), Dock bounce/badge attention signals, window-recreate-on-activate, the native app menu, and the Template tray icon. The darwin-gated code paths (trayIcon, TaskbarAttention, trayUpdateState/refreshTrayUpdateMenu, update-notice.tsx's darwin branch) are consistently and correctly guarded with `process.platform === 'darwin'` checks and have matching test coverage. One change breaks that pattern: the new native app menu is installed unconditionally on every platform, not just macOS, which is the one finding serious enough to block. Two smaller correctness/robustness gaps (an unguarded `undefined` interpolation in the mac update-notice label, and a redundant `Object.hasOwn` duplication) round out the findings.

## Critical Issues

### CR-01: Native macOS app menu is installed unconditionally on all platforms, not gated to darwin

**File:** `apps/desktop/main/index.ts:1128-1145`
**Issue:** `Menu.setApplicationMenu(Menu.buildFromTemplate([...]))` is called with no `process.platform === 'darwin'` guard, even though the comment directly above it says "Native macOS chrome (UI-03)" and the phase/requirement (UI-03) scopes this to macOS. Every other platform-specific addition in this same diff (`trayIcon()`, `TaskbarAttention`'s dock branch, `trayUpdateState`/`refreshTrayUpdateMenu`'s darwin branch, `update-notice.tsx`'s darwin branch) is explicitly gated on `process.platform === 'darwin'`; this call is not, breaking the established pattern.

This replaces Electron's default application menu on Windows/Linux with a template built almost entirely from macOS-only roles (`appMenu`, `about`, `services`, `hide`, `hideOthers`, `unhide` are documented by Electron as having no effect outside macOS) and a `Cmd+,` accelerator for Preferences that cannot fire on Windows/Linux (there is no "Cmd" key off macOS). The net effect on non-mac platforms is: the previous default menu (Reload, Toggle DevTools, Zoom, Quit, etc., reachable via Alt) is silently discarded and replaced with a menu that is mostly inert. Nothing in `tests/app-menu-wiring.test.mjs` (or elsewhere) asserts this call is darwin-gated — the test only checks the template's contents, not when it's applied — so this regression is untested and will ship.

**Fix:**
```ts
if (process.platform === 'darwin') {
  Menu.setApplicationMenu(Menu.buildFromTemplate([
    { role: 'appMenu', submenu: [ /* ... */ ] },
    { role: 'editMenu' },
    { role: 'windowMenu' },
  ]))
}
```
Add a corresponding assertion in `tests/app-menu-wiring.test.mjs` that the `Menu.setApplicationMenu(...)` call site is inside a `process.platform === 'darwin'` guard, mirroring how `tray-icon-wiring.test.mjs` and `taskbarattention.test.mjs` pin their darwin branches.

## Warnings

### WR-01: macOS "View release" button can render the literal string "undefined" as the version

**File:** `apps/desktop/renderer/src/update-notice.tsx:83-88`
**Issue:** The darwin branch interpolates `status.version` directly: `` `Orgtree ${status.version} available — View release` ``. `UpdateStatus.version` is optional (`version?: string`), and the comment block immediately above this component (lines 30-44) documents a real, reachable race where `version` is unset at `pending-idle` ("a fast cached autoDownload can fire `update-downloaded` before the check's own promise settles"). The non-mac branch already guards against exactly this case via `updateActionTitle(status.version)`, which falls back to a version-less string — the mac branch was added without reusing that same guard, so hitting the documented race renders "Orgtree undefined available — View release" to the user. No test in `update-notice-ui.test.mjs` covers `pending-idle` with `version` unset on darwin (the two darwin tests both hardcode `version: '2.0.5'`).
**Fix:**
```tsx
? <button className="update-now glow" title="View release" onClick={...}>
    {status.version ? `Orgtree ${status.version} available — View release` : 'A new version is available — View release'}
  </button>
```
Add a test mirroring the existing darwin tests but with `{ state: 'pending-idle' }` (no `version`) to lock in the fallback text.

### WR-02: `TaskbarAttention.start()`'s macOS branch re-bounces on every new arrival while already bouncing, orphaning the previous `bounceId`

**File:** `apps/desktop/main/taskbar-attention.ts:66-73`
**Issue:** `set()` calls `start()` whenever a genuinely new id arrives, with no check for `this.flashing` already being `true`. On darwin, this means `dock.bounce('critical')` is called again (a second native call) and `this.bounceId` is overwritten with the new return value, discarding the previous one without ever cancelling it. The file's own test suite (`tests/taskbarattention.test.mjs:136-143`) documents this is intentional ("a genuinely new arrival after a full drain bounces again with a new id") but only exercises the drain-then-return case, not the "still bouncing, second distinct id arrives" case — so the overwrite-while-active path (where the old id is discarded while still theoretically live) is unverified. If `dock.bounce('critical')` ever returns an id that must be individually cancelled (rather than the whole-app single-animation model), this leaks a handle. Low risk given Electron's actual dock-bounce semantics, but the code has no comment explaining why re-bouncing over an already-active bounce is safe, unlike every other branch in this file which is heavily commented.
**Fix:** Either add a `if (this.flashing) { this.bounceId = dock.bounce('critical'); return true }` early-return with a comment explaining the id-overwrite is safe, or explicitly skip re-calling `bounce()` when already flashing (`if (this.flashing) return false`) if the intent is genuinely to leave an in-progress bounce alone. Add a test for "new id arrives while already bouncing" to pin whichever behavior is chosen.

## Info

### IN-01: `viewReleaseVisible` derives from `status.state === 'pending-idle'` while the sibling `installVisible` derives from the separate `downloaded` boolean

**File:** `apps/desktop/main/updater.ts:439-454`
**Issue:** `installVisible`/`installEnabled` (used on non-mac) are keyed off the `downloaded` boolean passed in by the caller, but `viewReleaseVisible` (used on mac) is keyed off `status.state === 'pending-idle'` instead. In current usage (`apps/desktop/main/index.ts`) both are set synchronously in the same `update-downloaded` handler so they don't currently diverge, but the two mac/non-mac code paths now use two different signals to answer the same question ("is there a downloaded update ready to install"), which is a latent inconsistency if either signal's update timing ever changes independently.
**Fix:** Consider deriving `viewReleaseVisible` from `downloaded && !busy` (matching `ready`'s condition) rather than a second, independent check on `status.state`, so both platform branches read one shared signal.

---

_Reviewed: 2026-09-23T00:00:00Z_
_Reviewer: Claude (gsd-code-reviewer)_
_Depth: standard_
