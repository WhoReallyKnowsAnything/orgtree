---
phase: 04-macos-ui-os-integration-parity
plan: 01
subsystem: ui
tags: [electron, ipc, macos, dock, tray, update-notice, contracts]

# Dependency graph
requires:
  - phase: 01-packaging-runtime-foundation
    provides: a working, code-signed macOS package build to run this UI/OS-integration work against
provides:
  - "DesktopBridge.platform / DesktopBridge.openReleasePage IPC surface (packages/contracts, preload, main)"
  - "macOS update notice (in-app + tray) that never offers auto-install, both resolving the same MANUAL_UPGRADE_URL"
  - "TaskbarAttention dock-bounce branch on darwin, sharing the existing cross-org identity set"
  - "app.setBadgeCount wired to the same identity set as the dock bounce"
affects: [04-02-macos-ui-os-integration-parity]

# Actuals (#2632)
actuals:
  tokens: 9080
  tasks: 3
  commits: 3

# Tech tracking
tech-stack:
  added: []
  patterns:
    - "Bridge platform flag: DesktopBridge.platform (a plain process.platform string, not an IPC round trip) lets the renderer branch on OS without a new call per check"
    - "One implementation, two entry points: mac update copy/URL is computed once and consumed by both the renderer (via IPC) and the tray (main-process shell.openExternal directly) — never forked"
    - "Constructor-injected OS accessor (TaskbarAttention's `dock` param, mirroring the existing `target` param) instead of a top-level `import { app } from 'electron'`, so unit tests keep running outside a real Electron process via esbuild+plain node:test"

key-files:
  created: []
  modified:
    - packages/contracts/index.ts
    - apps/desktop/preload/index.ts
    - apps/desktop/main/index.ts
    - apps/desktop/main/updater.ts
    - apps/desktop/main/taskbar-attention.ts
    - apps/desktop/renderer/src/update-notice.tsx
    - tests/update-notice-ui.test.mjs
    - tests/updater.test.mjs
    - tests/taskbarattention.test.mjs
    - tests/electron.probe.ts

key-decisions:
  - "trayUpdateState()/refreshTrayUpdateMenu() take platform: string = process.platform rather than reading process.platform internally, keeping them pure/testable functions (matches the plan's exact signature)"
  - "On darwin, trayUpdateState() never populates installVisible/installEnabled (forces them false) rather than leaving them at their prior values — mac must never offer auto-install, so the field must actively say so"
  - "TaskbarAttention keeps using its existing `flashing` boolean as the generic active-signal gate for both platforms (set true/false in both start() branches), rather than adding a second darwin-only state flag — one gate, not two"

patterns-established:
  - "withPlatform(value, fn) test helper (Object.defineProperty override + restore), reused from tests/resolve-packaged-python-path.test.mjs, applied to two more test files to keep platform-branched unit tests deterministic on any host OS"

requirements-completed: [UPD-01, UI-01, UI-04]

coverage:
  - id: D1
    description: "macOS pending-idle update shows 'Orgtree {version} available — View release' (never an auto-apply 'Update now') in both the in-app notice and the tray menu, opening MANUAL_UPGRADE_URL via one shared implementation; a failed open shows the documented fallback copy"
    requirement: "UPD-01"
    verification:
      - kind: unit
        ref: "tests/update-notice-ui.test.mjs#darwin pending-idle offers View release, never an auto-apply Update now button"
        status: pass
      - kind: unit
        ref: "tests/update-notice-ui.test.mjs#darwin View release opens the release page, and reports the documented fallback on failure"
        status: pass
      - kind: unit
        ref: "tests/update-notice-ui.test.mjs#a plain-browser/back-compat bridge with no platform field still renders the existing Update now button unchanged"
        status: pass
      - kind: unit
        ref: "tests/updater.test.mjs#on darwin, the tray shows a View release row instead of Update now, toggled by pending-idle"
        status: pass
      - kind: unit
        ref: "tests/updater.test.mjs#trayUpdateState never populates installVisible/installEnabled on darwin - mac must never offer auto-install"
        status: pass
    human_judgment: false
  - id: D2
    description: "Dock icon bounces 'critical' for a qualifying-identity arrival, does not restart on re-poll or on one-of-several resolving, and stops when the qualifying set drains to empty; a later genuinely-new arrival bounces again"
    requirement: "UI-01"
    verification:
      - kind: unit
        ref: "tests/taskbarattention.test.mjs#darwin: a new arrival bounces the dock, a later poll of the same id does not re-bounce"
        status: pass
      - kind: unit
        ref: "tests/taskbarattention.test.mjs#darwin: the set draining to empty cancels the bounce with the stored id"
        status: pass
      - kind: unit
        ref: "tests/taskbarattention.test.mjs#darwin: an id seen again on a later poll after a full drain-and-return bounces again with a new id"
        status: pass
    human_judgment: false
  - id: D3
    description: "Dock badge count always equals the same qualifying identity-set length driving the bounce, from one shared handler invocation"
    requirement: "UI-04"
    verification:
      - kind: unit
        ref: "grep: apps/desktop/main/index.ts desktop:pending-attention handler computes `ids` once and passes it to both taskbarAttention.set(ids) and app.setBadgeCount(ids.length)"
        status: pass
    human_judgment: false

duration: ~50min
completed: 2026-09-23
status: complete
---

# Phase 4 Plan 1: macOS Update Notice, Tray Mirror, Dock Bounce + Badge Summary

**Wired the platform-flag/openReleasePage IPC bridge end-to-end on the mac update notice (in-app + tray, one implementation), then extended the existing cross-org attention identity set to drive both the Dock bounce and badge count on macOS.**

## Performance

- **Duration:** ~50 min
- **Completed:** 2026-09-23
- **Tasks:** 3/3 completed
- **Files modified:** 10 (6 in `files_modified` frontmatter + 3 companion test files touched by the same tasks + 1 out-of-scope regression fix in `tests/electron.probe.ts`)

## Accomplishments
- macOS never offers an auto-apply install: both the in-app `UpdateNotice` and the tray menu now render a "View release" action instead, resolving the exact same `MANUAL_UPGRADE_URL` constant through one code path each (renderer via new `desktop:open-release-page` IPC, tray via direct `shell.openExternal`) — never a forked implementation.
- Dock bounce (`app.dock.bounce('critical')`/`cancelBounce`) is wired into the existing `TaskbarAttention` class via a constructor-injected `dock` accessor, keeping the class's proven arrival/known-set identity logic completely untouched and platform-agnostic.
- Dock badge (`app.setBadgeCount`) and dock bounce now read the exact same `ids` array from the same `desktop:pending-attention` handler invocation — structurally impossible for them to drift apart.

## Task Commits

Each task was committed atomically:

1. **Task 1: End-to-end mac update notice — bridge platform flag → openReleasePage IPC → renderer branch** - `9e18cf0` (feat, tracer/tdd)
2. **Task 2: Mirror the mac update notice into the tray menu (one implementation, two entry points)** - `5382285` (feat)
3. **Task 3: Dock bounce + badge share one identity set (UI-01, UI-04)** - `f349785` (feat)

**Plan metadata:** SUMMARY commit follows this file's creation (see final_commit step).

## Files Created/Modified
- `packages/contracts/index.ts` - `DesktopBridge.platform: string` (first, non-async field) and `DesktopBridge.openReleasePage?()`
- `apps/desktop/preload/index.ts` - exposes `platform: process.platform` and `openReleasePage` IPC invoke
- `apps/desktop/main/index.ts` - `desktop:open-release-page` handler; darwin branch in `rebuildTray()`'s `updateRows`; `taskbarAttention` constructed with a `dock` accessor; `desktop:pending-attention` handler now also calls `app.setBadgeCount(ids.length)`
- `apps/desktop/main/updater.ts` - `trayUpdateState()`/`refreshTrayUpdateMenu()` take a `platform` parameter and compute `viewReleaseVisible`/`viewReleaseLabel` on darwin
- `apps/desktop/main/taskbar-attention.ts` - `dock` accessor constructor param; `start()`/`stop()` branch on `process.platform === 'darwin'`
- `apps/desktop/renderer/src/update-notice.tsx` - darwin render branch ahead of the existing `Update now` ternary arm
- `tests/update-notice-ui.test.mjs` - 3 new darwin/back-compat cases
- `tests/updater.test.mjs` - 2 new darwin tray cases; existing win32-oriented cases pinned to `'win32'` explicitly (see Deviations)
- `tests/taskbarattention.test.mjs` - `withPlatform()` helper; 4 new darwin dock cases; existing flash-based cases pinned to `'win32'` explicitly (see Deviations)
- `tests/electron.probe.ts` - two `refreshTrayUpdateMenu()` calls pinned to `'win32'` explicitly (see Deviations); not in this plan's `files_modified` list, but exercises the function Task 2 changed

## Decisions Made
- `trayUpdateState()`/`refreshTrayUpdateMenu()` gained `platform: string = process.platform` as a fifth parameter exactly as the plan specified, keeping them pure functions callers can pass an explicit platform into — this is what made the win32-pinning fix (below) possible without touching the win32/production call sites in `index.ts`.
- On darwin, `trayUpdateState()` actively forces `installVisible`/`installEnabled` to `false` rather than merely omitting them, so a caller reading the return value can never see a stale `true` from before the branch existed.
- Kept `TaskbarAttention`'s single `flashing` boolean as the "signal is active" gate for both the win32 (`flashFrame`) and darwin (`dock.bounce`) branches, so `stop()`'s existing `if (!this.flashing) return` guard needed no restructuring — only the branch bodies changed.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] `trayUpdateState`'s new `platform: string = process.platform` default silently broke every pre-existing win32-oriented test that omitted the parameter**
- **Found during:** Task 2, running `npm test` on `tests/updater.test.mjs`
- **Issue:** This worktree's real `process.platform` is `darwin` (a genuine macOS dev machine, consistent with this being the macOS port). Every existing test/probe call site that called `trayUpdateState(...)`/`refreshTrayUpdateMenu(...)` with fewer than 5 arguments picked up the new darwin-branch default and stopped exercising the win32 install-row behaviour those tests were actually asserting — 2 test failures.
- **Fix:** Added an explicit `'win32'` platform argument to every pre-existing call site whose assertions depend on `installVisible`/`installEnabled` (label-only/`checkEnabled`-only assertions are platform-independent and were left unchanged).
- **Files modified:** `tests/updater.test.mjs`, `tests/electron.probe.ts` (not in this plan's `files_modified`, but directly exercises the changed function)
- **Verification:** `node --test tests/updater.test.mjs` — 83/83 pass; `npm run typecheck` clean.
- **Committed in:** `5382285` (part of Task 2 commit)

**2. [Rule 1 - Bug] `TaskbarAttention`'s new darwin branch silently broke every pre-existing `flashFrame`-based test on this real macOS dev machine**
- **Found during:** Task 3, before running tests (identified proactively from the Task 2 pattern above, then confirmed by running the suite)
- **Issue:** Same root cause as #1: `start()`/`stop()` now branch on the real `process.platform`, which is `darwin` here, so every pre-existing test that constructed `new TaskbarAttention(() => w)` (no `dock` accessor, implicitly assuming win32 `flashFrame` behaviour) would silently take the new dock branch, find no dock, and never call `flashFrame` — none of those assertions would have exercised the code path they claim to test.
- **Fix:** Added a `withPlatform(value, fn)` helper (mirroring the existing one in `tests/resolve-packaged-python-path.test.mjs`) and wrapped every pre-existing `flashFrame`-based test body in `withPlatform('win32', () => {...})`, keeping the platform-independent `identities are validated at the boundary` test unwrapped. Added 4 new `withPlatform('darwin', ...)` tests for the dock-bounce acceptance criteria.
- **Files modified:** `tests/taskbarattention.test.mjs`
- **Verification:** `node --test tests/taskbarattention.test.mjs` — 11/11 pass; `npm run typecheck` clean.
- **Committed in:** `f349785` (part of Task 3 commit)

---

**Total deviations:** 2 auto-fixed (both Rule 1 - bugs directly caused by this plan's own platform-default changes)
**Impact on plan:** Both fixes were necessary for correctness — without them, this plan would have shipped tests that silently stopped testing the behaviour they claimed to, on the exact class of machine (macOS) this phase targets. No scope creep beyond the one incidental file (`tests/electron.probe.ts`) that exercises `refreshTrayUpdateMenu()` outside this plan's declared `files_modified`.

## Issues Encountered
None beyond the deviations above.

## User Setup Required
None - no external service configuration required.

## Out-of-Scope / Deferred

The full `npm test` suite (all `tests/*.test.mjs` files, not just this plan's three) surfaces pre-existing failures in `tests/update-rehearsal.test.mjs` unrelated to this plan — Windows-style test-fixture paths (`D:\...`, `E:\...`) whose error-message assertions don't hold in this environment. Neither this plan nor any file it modifies touches that test file or `tools/rehearsal-isolation.mjs`. Logged in full detail at `.planning/phases/04-macos-ui-os-integration-parity/deferred-items.md`; not fixed, per the executor's scope boundary. This plan's own verification instead ran its three specific test files (109/109 pass) plus `npm run typecheck` (clean), both directly tied to the `<verify>`/`<fails_when>` criteria in 04-01-PLAN.md.

## Next Phase Readiness
The `DesktopBridge.platform`/IPC-bridge pattern and the `process.platform === 'darwin'` gating convention established here (matching the pre-existing `win32` gating style in `apps/desktop/main/index.ts`) are ready for Plan 04-02's remaining OS-chrome work (application menu, window lifecycle, tray Template icon) to build on directly. No blockers.

---
*Phase: 04-macos-ui-os-integration-parity*
*Completed: 2026-09-23*

## Self-Check: PASSED

All 10 modified/created source and test files confirmed present on disk; all 3 task commits (`9e18cf0`, `5382285`, `f349785`) confirmed present in `git log`.
