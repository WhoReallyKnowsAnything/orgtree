---
phase: 04-macos-ui-os-integration-parity
plan: 02
subsystem: ui
tags: [electron, macos, window-lifecycle, app-menu, tray, template-image]

# Dependency graph
requires:
  - phase: 04-macos-ui-os-integration-parity
    plan: "04-01"
    provides: "the DesktopBridge.platform IPC surface and the tray's existing update-check/rebuildTray conventions this plan reuses"
provides:
  - "createMainWindow() shared construction helper — the single window-build path startup and show()'s recreate branch both call"
  - "Menu.setApplicationMenu (Orgtree/Edit/Window) — Preferences… broadcasts open-settings; Check for Updates… shares the tray's own checkForUpdates()"
  - "trayIcon() — the darwin-only Template-image wrapper around runtimeIcon(), used only by the two tray-specific call sites"
affects: []

# Actuals (#2632)
actuals:
  tokens: 9050
  tasks: 3
  commits: 4

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Outer-scope hoisted closure binding: createMainWindow is declared `let` in the outer block (next to placement) and assigned once inside app.whenReady(), so a helper defined in a narrower scope (closing over browserSession/initialOrigin/register/openArtifact) is still callable from show(), which lives in the outer block — one construction path, not a duplicated literal"
    - "One implementation, two entry points: the App Menu's Check for Updates… click handler is the exact same handler body text as the tray's own update-check row(s), never a forked second implementation"
    - "Source-level wiring tests for code that cannot be executed in isolation: index.ts requires Electron and App.tsx has no render-test harness for bridge.onEvent listeners, so tests/window-recreate.test.mjs, tests/app-menu-wiring.test.mjs and tests/tray-icon-wiring.test.mjs all assert wiring via regex against the source text, matching the codebase's own established idiom (lifetime-wiring.test.mjs, updater-wiring.test.mjs)"

key-files:
  created:
    - tests/window-recreate.test.mjs
    - tests/app-menu-wiring.test.mjs
    - tests/tray-icon-wiring.test.mjs
  modified:
    - apps/desktop/main/index.ts
    - packages/contracts/index.ts
    - apps/desktop/renderer/src/App.tsx
    - tests/icon-assets.test.mjs

key-decisions:
  - "show() is now async, guarding `!main || main.isDestroyed()` and awaiting createMainWindow() before falling through to its unchanged show/restore/maximize/focus/broadcast body — never a second construction path"
  - "createMainWindow is hoisted to an outer-scope `let` binding rather than inlined in both call sites, since browserSession/initialOrigin/register/openArtifact only exist inside app.whenReady()'s callback while show() lives in the outer block (the plan's own fallback guidance for this exact scope mismatch)"
  - "'open-settings' is appended AFTER 'open-org' in DesktopEvent['type'], not inserted before it, because tests/traylist-wiring.test.mjs pins 'popout-state' | 'open-org' as directly adjacent"
  - "rebuildTray() now makes two separate runtimeIcon()/trayIcon() calls instead of sharing one `image` binding across the tray and all windows — see Deviations"
  - "Every fire-and-forget show() call site (notification click, tray double-click, second-instance handler, openOrgFromTray, startup) was left as a bare, un-`void`-prefixed async call, matching the two call sites tests/traylist-wiring.test.mjs already pins literally — see Deviations"

patterns-established:
  - "Regex-based click-handler equality proof: rather than trying to prove function reference identity from static source text, the App Menu's Check for Updates… test counts occurrences of the exact handler-body string across the file and asserts the count equals (pre-existing tray occurrences + 1) — proving reuse without a forked copy, the same way updater-wiring.test.mjs proves updater.check() has exactly one call site"

requirements-completed: [UI-02, UI-03, UI-05]

coverage:
  - id: D1
    description: "Destroying the main window (main.destroy()) and then triggering activate/show() recreates exactly one window through the identical construction path used at startup; existing show()-while-alive restore/maximize/focus/broadcast behavior is unchanged"
    requirement: "UI-02"
    verification:
      - kind: unit
        ref: "tests/window-recreate.test.mjs#show() guards a destroyed/missing main window and recreates it before falling through"
        status: pass
      - kind: unit
        ref: "tests/window-recreate.test.mjs#exactly one main-window construction literal exists (startup and recreate share it)"
        status: pass
    human_judgment: false
  - id: D2
    description: "A native macOS App Menu (Orgtree/Edit/Window, no File/Help) exists; Preferences… (Cmd+,) opens the existing Settings panel via IPC broadcast; Check for Updates… shares the tray's exact checkForUpdates() function; Quit Orgtree quits with no confirmation dialog"
    requirement: "UI-03"
    verification:
      - kind: unit
        ref: "tests/app-menu-wiring.test.mjs#a native App Menu exists: Orgtree/Edit/Window, no File/Help"
        status: pass
      - kind: unit
        ref: "tests/app-menu-wiring.test.mjs#Preferences… broadcasts open-settings; Check for Updates… calls the tray's own checkForUpdates"
        status: pass
      - kind: unit
        ref: "tests/app-menu-wiring.test.mjs#App.tsx's open-settings listener toggles the existing Settings surface"
        status: pass
    human_judgment: false
  - id: D3
    description: "Edit menu's Undo/Redo/Cut/Copy/Paste/Select All enable/disable state self-manages via the built-in editMenu role's native focus tracking, with no custom code"
    requirement: "UI-03"
    verification:
      - kind: manual
        ref: "role: 'editMenu' used wholesale in the App Menu template — no custom enable/disable code exists to verify"
        status: unknown
    human_judgment: true
  - id: D4
    description: "The tray icon inverts automatically between light and dark macOS menu bars via a Template image, while the Dock icon and window icons keep their full-color branding completely unchanged"
    requirement: "UI-05"
    verification:
      - kind: unit
        ref: "tests/tray-icon-wiring.test.mjs#trayIcon() branches on darwin and only sets Template mode in that branch"
        status: pass
      - kind: unit
        ref: "tests/tray-icon-wiring.test.mjs#only the two tray-specific call sites use trayIcon(); window icons stay on runtimeIcon()"
        status: pass
      - kind: manual
        ref: "run the packaged app on macOS in light and dark menu bars — no automated test surface for actual menu-bar rendering"
        status: unknown
    human_judgment: true
  - id: D5
    description: "The provider-color tradeoff (Template images cannot carry the per-provider tray recolor) has been explicitly confirmed by the developer, or a tray.setTitle() follow-up has been explicitly requested instead"
    requirement: "UI-05"
    verification:
      - kind: manual
        ref: "see 'Open Decision Requiring Developer Confirmation' below — NOT yet confirmed by a human"
        status: unknown
    human_judgment: true

duration: ~45min
completed: 2026-09-23
status: complete
---

# Phase 4 Plan 2: Window Recreate, Native App Menu, Tray Template Icon Summary

**Closing the last Orgtree window no longer strands the Dock icon (show() recreates through the same startup construction path), Orgtree gained a real macOS App Menu whose Preferences/Check-for-Updates reuse existing conventions, and the tray glyph is now a Template image that auto-inverts for light/dark menu bars.**

## Performance

- **Duration:** ~45 min
- **Completed:** 2026-09-23
- **Tasks:** 3/3 completed (Task 2 via TDD: RED then GREEN)
- **Files modified:** 4 (`apps/desktop/main/index.ts`, `packages/contracts/index.ts`, `apps/desktop/renderer/src/App.tsx`, `tests/icon-assets.test.mjs`)
- **Files created:** 3 test files (`tests/window-recreate.test.mjs`, `tests/app-menu-wiring.test.mjs`, `tests/tray-icon-wiring.test.mjs`)

## Accomplishments

- `show()` is now async and guards `!main || main.isDestroyed()`, recreating the window via a shared `createMainWindow()` helper (hoisted to an outer-scope binding, assigned once inside `app.whenReady()`) before falling through to its unchanged show/restore/maximize/focus/broadcast body. Clicking the Dock icon after closing every window now always opens exactly one window instead of silently doing nothing.
- `Menu.setApplicationMenu` adds exactly three top-level entries (`appMenu`/`editMenu`/`windowMenu`, no File/Help). Preferences… (Cmd+,) broadcasts `{ type: 'open-settings' }`, picked up by a new `App.tsx` listener that reuses the existing `toggleSurface('org-settings', ...)` mechanism the Settings-gear button already uses. Check for Updates… uses the exact same click-handler body as the tray's own update-check row(s) — one implementation, several entry points.
- `trayIcon()` wraps `runtimeIcon()` on darwin: zeroes the B/G/R bytes of every pixel (keeping alpha), rebuilds via `nativeImage.createFromBitmap`, and calls `setTemplateImage(true)`. Only the two tray-specific call sites (`new Tray(...)`, `rebuildTray()`'s tray image) use it — the Dock icon and every window icon stay on the unwrapped, full-color `runtimeIcon()`.

## Task Commits

Each task was committed atomically:

1. **Task 1: Window lifecycle — recreate a destroyed main window on show() (UI-02)**
   - Commit: `9152550` — `feat(04-02): recreate destroyed main window on show() (UI-02)`
   - Files: `apps/desktop/main/index.ts`, `tests/window-recreate.test.mjs`

2. **Task 2: Native macOS app menu (UI-03)** — TDD RED then GREEN
   - RED commit: `4c46d52` — `test(04-02): add failing test for native app menu (UI-03)`
   - GREEN commit: `02cf637` — `feat(04-02): implement native app menu (UI-03)`
   - Files: `apps/desktop/main/index.ts`, `packages/contracts/index.ts`, `apps/desktop/renderer/src/App.tsx`, `tests/app-menu-wiring.test.mjs`

3. **Task 3: Tray Template icon (UI-05)**
   - Commit: `dda05f5` — `feat(04-02): tray Template icon inverts for light/dark menu bars (UI-05)`
   - Files: `apps/desktop/main/index.ts`, `tests/icon-assets.test.mjs`, `tests/tray-icon-wiring.test.mjs`

## TDD Gate Compliance

Task 2 was authored with `tdd="true"`. Gate sequence confirmed in git log:
1. RED — `test(04-02): add failing test for native app menu (UI-03)` (`4c46d52`), all 5 assertions confirmed failing via `node --test` before the commit.
2. GREEN — `feat(04-02): implement native app menu (UI-03)` (`02cf637`), all 5 assertions confirmed passing via `node --test` before the commit.
3. REFACTOR — not needed; no changes made after GREEN.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] `rebuildTray()`'s shared `image` binding would have made every window icon monochrome**

- **Found during:** Task 3, before committing (caught by re-reading the actual source, not by the plan's own `read_first` note)
- **Issue:** The plan's `read_first`/`action` described `main.setIcon(runtimeIcon())`/`viewer.setIcon(runtimeIcon())`/`child.setIcon(runtimeIcon())` as the only window-icon call sites, separate from `rebuildTray()`. In the actual code, `rebuildTray()` has a FOURTH icon call site: `const image = runtimeIcon(); tray?.setImage(image); for (const window of BrowserWindow.getAllWindows()) window.setIcon(image)` — one shared binding feeding both the tray AND every open window. Following the plan's literal instruction ("swap rebuildTray()'s tray-image assignment to trayIcon()") by simply reassigning that one binding would have made the Dock icon and every window icon monochrome on every tray rebuild (theme change, engine status change, etc.) — directly violating this plan's own must_have truth that window/Dock icons stay full-color.
- **Fix:** Split into two separate calls: `tray?.setImage(trayIcon())` for the tray, then `const image = runtimeIcon()` (name kept unchanged from before this task) still feeding the window loop unmodified.
- **Files modified:** `apps/desktop/main/index.ts`
- **Commit:** `dda05f5`

**2. [Rule 1 - Bug, self-corrected before commit] An initial `void`-wrapping pass broke two pre-existing pinned wiring tests**

- **Found during:** Task 1, before committing
- **Issue:** Making `show()` async left five of its call sites as unawaited (fire-and-forget) calls. An initial pass wrapped all five in `void` for stylistic consistency with the file's other ignored-promise call sites. `npm test` then failed two pre-existing assertions in `tests/traylist-wiring.test.mjs` that literally regex-match the `openOrgFromTray` and tray double-click call sites without `void`.
- **Fix:** Reverted all five `void` additions back to bare `show()` calls. Async/await inside `show()` needed no caller changes; the codebase already tolerates a floating promise at these exact call sites (proven by the pre-existing pinned tests).
- **Files modified:** `apps/desktop/main/index.ts`
- **Commit:** `9152550`

**3. [Rule 1 - Bug, self-corrected before commit] `DesktopEvent['type']` insertion point broke a pre-existing pinned test**

- **Found during:** Task 2, before committing
- **Issue:** Initially added `'open-settings'` between `'popout-state'` and `'open-org'` in the union. `tests/traylist-wiring.test.mjs` pins `'popout-state' | 'open-org'` as directly adjacent; inserting a member between them broke that regex match.
- **Fix:** Appended `'open-settings'` AFTER `'open-org'` instead — same net addition, no renumbering of the existing pinned sequence.
- **Files modified:** `packages/contracts/index.ts`
- **Commit:** `02cf637`

**4. [Rule 1 - Bug] Three pre-existing regex bugs in the new app-menu-wiring.test.mjs, found by running it against the real implementation**
- The `DesktopEvent` match required `'open-settings'` immediately before `'open-org'` with no ` | ` separator token — could never match as written.
- The appMenu submenu match did not account for the trailing comma after the submenu array literal (`],`).
- The `checkForUpdates` handler-body occurrence count assumed exactly 1 pre-existing tray call site; `rebuildTray()` actually has 2 (darwin and non-darwin branches), so the correct invariant is `trayOccurrences + 1`, not a hardcoded `2`.
- **Files modified:** `tests/app-menu-wiring.test.mjs`
- **Commit:** `02cf637`

**Total deviations:** 4 auto-fixed (2 bugs in production code, 2 self-corrected test-authoring mistakes) — all found and fixed before their respective task commits, via running the actual tests rather than by inspection alone.

### Pre-existing, out-of-scope test environment failures

`npm test`'s full run reports 51 failures unrelated to any file this plan modifies — real-ACL trust checks, Windows `.cmd`-wrapper process-tree spawning, and Windows rehearsal-path fixtures, none reachable from macOS window/menu/tray code. This exact baseline (same test names) was already documented as pre-existing and out of scope by `04-01-SUMMARY.md`/`deferred-items.md`. Verified via a failure-set diff (name-only, timing excluded) across every commit in this plan: zero new failures, zero fixed failures, at every step.

`npm run test:renderer` has one pre-existing, unrelated failure: `ENOENT: no such file or directory, open '.../src/styles.css'` in a provider-theme build script — a working-directory path bug in that script, not touched by this plan's `App.tsx` change.

## Open Decision Requiring Developer Confirmation

**This is NOT yet resolved. Do not treat UI-05 as fully closed until a human explicitly answers this.**

04-UI-SPEC.md flags a real, documented tradeoff: a macOS Template image is alpha-channel-only, so the existing per-provider tray recolor (`orgtree-eye-tray-{claude,codex,antigravity,openrouter,orgtree,grey}.ico`, applied via pixel recoloring in `runtimeIcon()`) becomes invisible in the menu bar once Template mode is on. This plan implements the UI-SPEC's own documented **default**: ship the single monochrome Template glyph, keep `tray.setToolTip()`'s existing provider/engine-state text as the sole carrier of that signal, and do not build a second image. That default is what got implemented — it was NOT chosen by asking a human; it was applied because it is the spec's stated default, and the plan's own `must_haves.truths` requires an explicit developer answer before this counts as done:

> "The provider-color tradeoff... has been explicitly confirmed by the developer before this plan is considered done, or a tray.setTitle() follow-up has been explicitly requested instead."

**Two paths forward, developer's call:**
1. **Confirm the default is acceptable for v1** — the tooltip/menu text carries the provider signal; the icon itself is monochrome black/white, auto-inverting for light/dark. No further code change needed.
2. **Request the `tray.setTitle(text)` follow-up** — macOS-only, renders text beside the icon in the system's own (already theme-correct) label color, restoring a visible provider signal in the menu bar itself. Not built in this plan; would be a new task.

Also unresolved by anything automatable: **D3** (Edit-menu enable/disable state) and the **light/dark visual check** in D4 both require a human running the packaged app on macOS — there is no automated test surface for native OS menu-bar/menu-item rendering.

## Self-Check: PASSED

- `apps/desktop/main/index.ts` — FOUND
- `packages/contracts/index.ts` — FOUND
- `apps/desktop/renderer/src/App.tsx` — FOUND
- `tests/window-recreate.test.mjs` — FOUND
- `tests/app-menu-wiring.test.mjs` — FOUND
- `tests/tray-icon-wiring.test.mjs` — FOUND
- `tests/icon-assets.test.mjs` — FOUND
- Commit `9152550` — FOUND in `git log --oneline`
- Commit `4c46d52` — FOUND in `git log --oneline`
- Commit `02cf637` — FOUND in `git log --oneline`
- Commit `dda05f5` — FOUND in `git log --oneline`
- `npm run typecheck` — clean (zero errors) after every task
- `npm test` — 668 tests, 589 pass, 51 fail (pre-existing baseline, zero new failures — verified by failure-set diff at every commit)
- `node --test tests/window-recreate.test.mjs tests/app-menu-wiring.test.mjs tests/tray-icon-wiring.test.mjs` — 13/13 pass
