---
phase: 04-macos-ui-os-integration-parity
fixed_at: 2026-09-23T12:45:04Z
review_path: .planning/phases/04-macos-ui-os-integration-parity/04-REVIEW.md
iteration: 1
findings_in_scope: 3
fixed: 3
skipped: 0
status: all_fixed
---

# Phase 04: Code Review Fix Report

**Fixed at:** 2026-09-23T12:45:04Z
**Source review:** .planning/phases/04-macos-ui-os-integration-parity/04-REVIEW.md
**Iteration:** 1

**Summary:**
- Findings in scope: 3 (1 critical, 2 warning; IN-01 skipped per fix scope)
- Fixed: 3
- Skipped: 0

## Fixed Issues

### CR-01: Native macOS app menu is installed unconditionally on all platforms, not gated to darwin

**Files modified:** `apps/desktop/main/index.ts`, `tests/app-menu-wiring.test.mjs`
**Commit:** 4d80b5c
**Applied fix:** Wrapped `Menu.setApplicationMenu(Menu.buildFromTemplate([...]))` in `if (process.platform === 'darwin') { ... }`, matching the darwin-gating pattern already used for `trayIcon()`, `TaskbarAttention`'s dock branch, and `trayUpdateState`/`refreshTrayUpdateMenu` in the same file. Non-darwin platforms now keep Electron's default application menu instead of getting the mostly-inert macOS-role template. Added a regression test (`Menu.setApplicationMenu is only installed on darwin`) asserting the call site is inside the guard.

### WR-01: macOS "View release" button can render the literal string "undefined" as the version

**Files modified:** `apps/desktop/renderer/src/update-notice.tsx`, `tests/update-notice-ui.test.mjs`
**Commit:** 0b85c08
**Applied fix:** Changed the darwin branch's button label from an unconditional `` `Orgtree ${status.version} available — View release` `` to a ternary that falls back to `'A new version is available — View release'` when `status.version` is unset, mirroring the guard the non-mac `updateActionTitle()` path already applies for the same documented race (a fast cached autoDownload can fire `update-downloaded` before the check's own promise settles). Added a test (`darwin pending-idle with no version falls back to a version-less label, never "undefined"`) covering `pending-idle` with no version on darwin.

### WR-02: `TaskbarAttention.start()`'s macOS branch re-bounces on every new arrival while already bouncing, orphaning the previous `bounceId`

**Files modified:** `apps/desktop/main/taskbar-attention.ts`, `tests/taskbarattention.test.mjs`
**Commit:** 0eb226a
**Applied fix:** Added an early-return guard (`if (this.flashing) return false`) at the top of the darwin branch of `start()`, with a comment explaining that a further arrival while already bouncing has nothing new to start and that re-calling `bounce()` would overwrite `bounceId` with a second native call's id, orphaning the first one. This matches the file's existing early-return style used for other "nothing further to do" conditions (e.g. the win32 branch's already-focused-window check). Chose the "skip re-bounce while flashing" option from the review's two suggested fixes, since it removes the orphaned-handle risk entirely rather than just documenting it. Added a test (`darwin: a genuinely new id while already bouncing does not re-bounce or orphan the id`) pinning this behavior; the pre-existing "full drain-and-return bounces again with a new id" test still passes unchanged since `flashing` is `false` after a full drain.

## Skipped Issues

None — all in-scope findings were fixed. IN-01 (`viewReleaseVisible`/`installVisible` signal inconsistency in `apps/desktop/main/updater.ts`) is Info severity and was excluded per the default fix scope (Critical + Warning only).

## Verification

- `node --test tests/app-menu-wiring.test.mjs tests/update-notice-ui.test.mjs tests/taskbarattention.test.mjs tests/tray-icon-wiring.test.mjs tests/window-recreate.test.mjs tests/updater.test.mjs` — 125 tests, 0 failures.
- `npm run typecheck` — clean, no errors, after each of the three fixes.
- All verification ran in the working directory (no isolated worktree was created; the caller specified the working directory was already on the correct branch/HEAD).

---

_Fixed: 2026-09-23T12:45:04Z_
_Fixer: Claude (gsd-code-fixer)_
_Iteration: 1_
