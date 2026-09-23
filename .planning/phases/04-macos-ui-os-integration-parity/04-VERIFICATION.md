---
phase: 04-macos-ui-os-integration-parity
verified: 2026-09-23T00:00:00Z
status: human_needed
score: 6/6 must-haves verified (code+test), 6 items routed to human verification
behavior_unverified: 1
overrides_applied: 0
re_verification: null
behavior_unverified_items:
  - truth: "Closing the last window and clicking the Dock icon (activate) recreates the main window via createMainWindow() exactly once — never zero, never two — after the previous window was fully destroyed."
    test: "Launch the packaged .app, close the only window (app stays in Dock), click the Dock icon once, confirm exactly one window opens; repeat twice to confirm no double-open and no dead click."
    expected: "Exactly one window opens on every click; the app never becomes an inert Dock icon that does nothing on click."
    why_human: "tests/window-recreate.test.mjs only pattern-matches the source (single createMainWindow() call site, shared by startup and show()); no test exercises a real Electron BrowserWindow being destroyed and recreated, since these test files run outside an Electron runtime by design (esbuild+node:test)."
human_verification:
  - test: "Trigger cross-org attention (an org needs a response) and watch the real Dock icon."
    expected: "Dock icon bounces continuously ('critical' style); a second still-waiting org does not restart or double the bounce; resolving all qualifying orgs stops the bounce."
    why_human: "app.dock.bounce()'s actual on-screen animation and the OS's own critical-bounce cadence cannot be observed via unit tests — only the TaskbarAttention state machine (start/stop/no-re-bounce) is exercised by tests/taskbarattention.test.mjs, not the native call's visible effect."
  - test: "Close the last Orgtree window, then click the Dock icon."
    expected: "Exactly one window opens (see behavior_unverified_items above)."
    why_human: "Real BrowserWindow destroy/recreate lifecycle, not reachable from tests/window-recreate.test.mjs's source-pattern assertions."
  - test: "Open the native macOS menu bar while Orgtree is frontmost."
    expected: "An Orgtree/Edit/Window menu bar (no File/Help); Preferences… (Cmd+,) opens Settings; Check for Updates… triggers the same check as the tray; Quit Orgtree quits without a confirmation dialog; Cmd+C/Cmd+V/Cmd+Z work in text fields."
    why_human: "tests/app-menu-wiring.test.mjs asserts source-level wiring (darwin gate, template structure, handler bodies) but cannot render or click a real macOS menu bar."
  - test: "Edit menu item enable/disable with no text field focused vs. a field focused."
    expected: "Undo/Redo/Cut/Copy/Paste/Select All grey out with nothing focused and activate once a text field has focus/selection, self-managed by the built-in `editMenu` role."
    why_human: "Explicitly flagged as a backstop in 04-02-PLAN.md must_haves — 'no automated test surface for native OS menu enablement'."
  - test: "Toggle the Mac menu bar between light and dark appearance while Orgtree's tray icon is visible."
    expected: "The tray icon inverts (white glyph on dark bar, black glyph on light bar) automatically; the Dock icon and all window icons stay full-color and unchanged."
    why_human: "tests/tray-icon-wiring.test.mjs proves trayIcon() builds a monochrome nativeImage and calls setTemplateImage(true) exactly once in the darwin branch, but cannot observe the OS's actual color-inversion rendering."
  - test: "Trigger a pending-idle update with an unusually long version string (or a long build/channel suffix) and view the in-app update notice."
    expected: "The notice text wraps inside the existing `.update-notice` container (`max-width: min(280px, 34vw); overflow-wrap: anywhere`) with no visual overflow or clipping."
    why_human: "Explicitly flagged as a backstop in 04-01-PLAN.md must_haves — 'verify by hand with a long version/build string at implementation time'."
---

# Phase 4: macOS UI & OS Integration Parity Verification Report

**Phase Goal:** Dock/menu/tray behavior and update notices match native macOS conventions — Orgtree looks and behaves like a native macOS app instead of a ported Windows app.
**Verified:** 2026-09-23
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths (Roadmap Success Criteria + PLAN must_haves)

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | Dock icon bounces (`app.dock.bounce('critical')`) instead of `flashFrame` on attention; no re-bounce on repeat id; resolved-among-several doesn't stop/restart; drain to empty stops it | ✓ VERIFIED (logic+test) | `apps/desktop/main/taskbar-attention.ts` L34-86: `set()`/`start()`/`stop()` state machine, darwin branch calls `dock.bounce('critical')`/`cancelBounce`. `tests/taskbarattention.test.mjs` — all darwin cases pass incl. new WR-02 regression test L145 ("a genuinely new id while already bouncing does not re-bounce or orphan the id"). Native visual bounce → human item. |
| 2 | Dock badge count always equals the same qualifying identity-set length as the bounce (one shared source of truth) | ✓ VERIFIED | `apps/desktop/main/index.ts` L1275-1283: both `taskbarAttention.set(ids)` and `app.setBadgeCount(ids.length)` read the same local `ids` const from one handler invocation — no second counter. |
| 3 | Closing the last window leaves the app in the Dock; clicking the Dock icon always reopens exactly one window, even after main was fully destroyed | ⚠️ PRESENT_BEHAVIOR_UNVERIFIED | `show()` (index.ts L251-254) guards `(!main \|\| main.isDestroyed())` and calls the same `createMainWindow` used at startup; `app.on('activate', show)` (L1126), `app.on('window-all-closed', noop)` (L1127). `tests/window-recreate.test.mjs` (4/4 pass) proves single-call-site source wiring only — no test exercises an actual Electron window destroy→recreate cycle. See behavior_unverified_items. |
| 4 | Native macOS app menu (Orgtree/Edit/Window) exists where none did; Preferences (Cmd+,) opens Settings; Check for Updates reuses the tray's function; Quit has no confirmation; Cmd+C/V/Z via editMenu role | ✓ VERIFIED (logic+test) | `index.ts` L1133-1158, gated `if (process.platform === 'darwin')` (CR-01 fix). `tests/app-menu-wiring.test.mjs` pins the darwin guard, the `open-settings` broadcast, and the shared `checkForUpdates` reference. Rendered menu bar → human item. |
| 5 | Tray icon is a Template image that auto-inverts light/dark; Dock icon and window icons stay full-color, unchanged | ✓ VERIFIED (logic+test) | `trayIcon()` (index.ts L228-236): darwin branch zeroes RGB, keeps alpha, calls `.setTemplateImage(true)` once; non-darwin returns `runtimeIcon()` unchanged. Only `rebuildTray()` and the initial `new Tray(...)` use it; `main/viewer/child.setIcon` still call `runtimeIcon()` directly. `tests/tray-icon-wiring.test.mjs` passes. Visual inversion → human item. |
| 6 | User sees a "new version available" notice (never auto-apply) on macOS; same `MANUAL_UPGRADE_URL` in-app and in tray; failed open shows documented fallback copy; unreachable check keeps unchanged label | ✓ VERIFIED (logic+test) | `update-notice.tsx` L80-88 (darwin branch, WR-01 unset-version guard present), `updater.ts` `trayUpdateState()` L419-455 (`viewReleaseVisible`/`viewReleaseLabel`, WR-01 guard at L420), `index.ts` L1287-1290 (`desktop:open-release-page` handler only ever opens the hardcoded `MANUAL_UPGRADE_URL`, never a renderer-supplied one). `tests/update-notice-ui.test.mjs` + `tests/updater.test.mjs`: 121/121 passing across both files plus taskbar/app-menu/tray-icon suites. |
| 7 | Provider-color tray tradeoff (Template images lose per-provider recolor) explicitly confirmed by the developer, or a `tray.setTitle()` follow-up explicitly requested instead | PASSED (override — see note) | `04-02-SUMMARY.md` L104-108 records this as **not yet confirmed** at summary time. Per this verification's task brief, the tradeoff was subsequently surfaced to and accepted by the developer out-of-band; treated as satisfied per explicit instruction, not re-litigated here. |

**Score:** 6/6 code-level truths verified and test-covered (2 with an additional native-runtime aspect still needing a human eyes-on pass, 1 behavior-dependent truth left PRESENT_BEHAVIOR_UNVERIFIED, 1 passed via documented override).

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `packages/contracts/index.ts` | `DesktopBridge.platform`, `DesktopBridge.openReleasePage`, `DesktopEvent` gains `'open-settings'` | ✓ VERIFIED | L94 `platform: string`, L130 `openReleasePage?()`; `DesktopEvent['type']` includes `'open-settings'` (confirmed by `tests/app-menu-wiring.test.mjs`). |
| `apps/desktop/preload/index.ts` | `bridge.platform`, `bridge.openReleasePage` | ✓ VERIFIED | L9 `platform: process.platform`, L32 `openReleasePage: () => ipcRenderer.invoke('desktop:open-release-page')`. |
| `apps/desktop/main/index.ts` | `desktop:open-release-page` handler; darwin app-menu; darwin `trayIcon()`; shared badge/bounce `ids`; `createMainWindow` reuse | ✓ VERIFIED | All confirmed by line-level reads above. |
| `apps/desktop/main/updater.ts` | darwin-aware `trayUpdateState()`/`refreshTrayUpdateMenu()` | ✓ VERIFIED | L419-455, L519-531. |
| `apps/desktop/main/taskbar-attention.ts` | darwin `dock.bounce`/`cancelBounce` branch, injected accessor | ✓ VERIFIED | Constructor takes `dock?: () => Dock`, darwin branches in `start()`/`stop()`; WR-02 guard present (L73 `if (this.flashing) return false`). |
| `apps/desktop/renderer/src/update-notice.tsx` | darwin render branch, unset-version guard, error fallback | ✓ VERIFIED | L80-88. |
| `apps/desktop/renderer/src/App.tsx` | `bridge.onEvent` listener for `'open-settings'` | ✓ VERIFIED | L563-570. |

### Key Link Verification

| From | To | Via | Status | Details |
|------|----|----|--------|---------|
| `update-notice.tsx` darwin branch | `shell.openExternal(MANUAL_UPGRADE_URL)` | `bridge.openReleasePage()` → `ipcRenderer.invoke` → `desktop:open-release-page` handler | ✓ WIRED | index.ts L1287-1290, never accepts a renderer-supplied URL. |
| `rebuildTray()`'s darwin update row | same `MANUAL_UPGRADE_URL` | `shell.openExternal` at L408, same constant imported L20 | ✓ WIRED | One implementation, two entry points confirmed — no forked copy found. |
| `TaskbarAttention.start()/stop()` darwin branch | `app.dock.bounce`/`cancelBounce` | constructor-injected `dock` accessor (`() => app.dock`, index.ts L245) | ✓ WIRED | Not a direct `electron` import — preserves esbuild+node:test runnability, confirmed by passing test suite. |
| App Menu "Preferences…" click | Settings panel opens | `broadcast({type:'open-settings'})` → `App.tsx` `bridge.onEvent` → `toggleSurface('org-settings', ...)` | ✓ WIRED | index.ts L1140, App.tsx L563-570. |
| App Menu "Check for Updates…" click | update check runs | same `checkForUpdates` function reference as tray row | ✓ WIRED | index.ts L1150 calls `checkForUpdates()`, same closure the tray's row (L~705) calls — comment at L1148-1149 confirms intent, no second implementation found. |
| `rebuildTray()`/`new Tray(...)` | monochrome Template icon | `trayIcon()` → `runtimeIcon()` | ✓ WIRED | index.ts L389, L1229; `main/viewer/child.setIcon` confirmed still on unwrapped `runtimeIcon()`. |
| `app.on('activate', show)` | window recreation | `show()` → `createMainWindow()` | ✓ WIRED (source-level; runtime unverified) | index.ts L251-252, L1126. |

### Behavioral Spot-Checks / Test Execution

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Full phase-4 test surface (dock/badge, update notice, tray, app menu, window recreate) | `node --test tests/taskbarattention.test.mjs tests/updater.test.mjs tests/update-notice-ui.test.mjs tests/app-menu-wiring.test.mjs tests/tray-icon-wiring.test.mjs` | 121 pass, 0 fail, 0 cancelled | ✓ PASS |
| Window-recreate wiring | `node --test tests/window-recreate.test.mjs` | 4 pass, 0 fail | ✓ PASS |
| Type safety | `npm run typecheck` (`tsc --noEmit`) | exit 0, no errors | ✓ PASS |
| WR-02 regression: new id while already bouncing does not orphan `bounceId` | `taskbarattention.test.mjs:145` ("darwin: a genuinely new id while already bouncing does not re-bounce or orphan the id") | pass | ✓ PASS |
| CR-01 regression: app menu only installed on darwin | `app-menu-wiring.test.mjs:56` ("Menu.setApplicationMenu is only installed on darwin") | pass | ✓ PASS |

### Probe Execution

SKIPPED — no `scripts/*/tests/probe-*.sh` probes declared or found for this phase; this is a source-level TS/Electron feature phase, not a migration/tooling phase.

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|--------------|--------|----------|
| UI-01 | 04-01 | Dock bounce replaces `flashFrame` on macOS | ✓ SATISFIED | taskbar-attention.ts darwin branch + passing tests. REQUIREMENTS.md already checked `[x]`/Complete. |
| UI-02 | 04-02 | Window lifecycle follows macOS convention (`window-all-closed`/`activate`) | ✓ SATISFIED (code); runtime behavior unverified | index.ts L1126-1127, L251-252 + window-recreate.test.mjs source assertions. **REQUIREMENTS.md still shows `[ ]` Pending — stale, see Gaps Summary.** |
| UI-03 | 04-02 | Native macOS app menu | ✓ SATISFIED | index.ts L1133-1158 (darwin-gated per CR-01) + app-menu-wiring.test.mjs. **REQUIREMENTS.md still shows `[ ]` Pending — stale.** |
| UI-04 | 04-01 | Dock badge count via `app.setBadgeCount()` | ✓ SATISFIED | index.ts L1275-1283, shared `ids` source. REQUIREMENTS.md already checked `[x]`/Complete. |
| UI-05 | 04-02 | Tray icon Template image for light/dark | ✓ SATISFIED | trayIcon() darwin branch + tray-icon-wiring.test.mjs. **REQUIREMENTS.md still shows `[ ]` Pending — stale.** |
| UPD-01 | 04-01 | "New version available" notice, no auto-apply | ✓ SATISFIED | update-notice.tsx + updater.ts + open-release-page handler, 121 passing tests. REQUIREMENTS.md already checked `[x]`/Complete. |

All 6 requirement IDs declared across both plans are accounted for in REQUIREMENTS.md; no orphaned requirements found for Phase 4.

### Anti-Patterns Found

None (blocker or warning level). No `TBD`/`FIXME`/`XXX`/`TODO`/`HACK`/`PLACEHOLDER` markers in any of the 7 files touched by this phase's plans. No hollow stubs, no hardcoded-empty renders, no console.log-only handlers.

**Housekeeping (info, not a blocker):** `.planning/REQUIREMENTS.md` traceability table (lines 84-87) still lists UI-02, UI-03, UI-05 as `Pending` / unchecked, even though the implementation, wiring, and tests for all three are verified present in the codebase (see Requirements Coverage above). This looks like a bookkeeping step that was missed when Phase 4 wrapped, not a code gap — recommend flipping those three checkboxes/status cells to match UI-01/UI-04/UPD-01's already-updated `[x]`/Complete state.

Code review (04-REVIEW.md) found 1 critical + 2 warnings; 04-REVIEW-FIX.md fixed all 3 (CR-01 darwin-gate, WR-01 unset-version guard, WR-02 re-bounce guard), each independently confirmed present in code and covered by a new regression test above. The 1 info-level finding (`IN-01`, `viewReleaseVisible` deriving from a different signal than `installVisible`) was left open by design (info severity, out of the fix-scope's 3 findings) — latent inconsistency, not a functional bug; no action required for phase-goal achievement.

### Human Verification Required

See `human_verification` in frontmatter for the full list (6 items): native Dock bounce animation, Dock-icon-click window recreation, native app-menu-bar rendering/behavior, Edit-menu native enable/disable, tray icon light/dark inversion, and long-version-string notice wrapping. All require the packaged `.app` running on real macOS — none can be proven by static analysis or the existing esbuild+node:test harness, which by design runs outside a real Electron runtime.

### Gaps Summary

No blocking gaps. Every roadmap Success Criterion and every PLAN must_have truth has a corresponding, currently-passing implementation and test in the codebase; all 3 code-review findings (1 critical, 2 warning) were fixed and each fix has its own regression test. The only items keeping this phase out of a clean `passed` are: (1) six OS-level visual/interactive behaviors that are inherently unverifiable outside a running packaged macOS app (dock bounce, menu-bar rendering, tray inversion, Dock-click window recreate, Edit-menu native enable/disable, long-string wrap) and (2) three stale checkboxes in REQUIREMENTS.md that don't match the (verified) actual implementation state. Neither blocks phase-goal achievement in the codebase; both are appropriate follow-ups — the human checks before considering the phase fully closed, and a one-line REQUIREMENTS.md edit.

---

_Verified: 2026-09-23_
_Verifier: Claude (gsd-verifier)_
