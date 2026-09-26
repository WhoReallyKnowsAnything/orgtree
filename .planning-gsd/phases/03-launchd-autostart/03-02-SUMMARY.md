---
phase: 03-launchd-autostart
plan: 02
subsystem: desktop-main
tags: [macos, launchd, autostart, electron]
status: complete
dependency-graph:
  requires: [03-01]
  provides: [darwin-autostart-wired]
  affects: [apps/desktop/main/index.ts]
tech-stack:
  added: []
  patterns:
    - "Best-effort auxiliary block after a successful primary action, wrapped in its own try/catch (console.warn on failure), never blocking or throwing into the caller's success path"
    - "Native remediation dialog -> shell.openExternal(fixed literal URL), mirroring existing showCrashReports wiring (index.ts:142-151)"
key-files:
  created: []
  modified:
    - apps/desktop/main/index.ts
decisions:
  - "Task 1 (confirm resolveMacEnginePythonPath's interpreter subpath) required no code change: 03-01 had already verified it against apps/desktop/main/engine.ts:77-79's resolvePackagedPythonPath (Phase 1's real darwin output, runtime/bin/python3.13), cross-checked here against .planning/phases/01-packaging-runtime-foundation/01-01-SUMMARY.md:50 which independently confirms the same path from provision-runtime.py's real run. No [ASSUMED] marker remained to replace."
  - "Inserted the darwin autostart branch immediately after the engine attach/start success point (the closing brace of the `if (!await engine.attach(...))` block), not at the plan's literal line 1300/1301 anchors — the file had shifted ~28 lines from concurrent phase work since the plan was written. The structural position (after successful attach/start, before `session.fromPartition`) is unchanged and was used as the anchor instead of stale absolute line numbers."
  - "Added a .catch(() => {}) to the disabled-state dialog's .then() chain (not in the plan's literal action text) to avoid an unhandled promise rejection, matching the existing pattern already used elsewhere in index.ts (e.g. the showUpdateFailure .then/.catch at line ~1607)."
metrics:
  duration: "12m"
  completed: 2026-09-23
actuals:
  tokens: 380
  tasks: 2
  commits: 1
---

# Phase 3 Plan 02: Wire darwin autostart into app startup Summary

Wired Plan 01's isolated `launchagent-mac.ts` module into `apps/desktop/main/index.ts`'s real darwin startup sequence: after a successful engine attach/start, the app now calls `detectState()` once, auto-installs the LaunchAgent when `not-installed`, and shows a native remediation dialog (with a button to System Settings > Login Items & Extensions) when `disabled`. No settings/GUI toggle was added.

## What Was Built

**Task 1 — Confirm packaged Python interpreter path (no change needed):** `resolveMacEnginePythonPath()` in `launchagent-mac.ts` already carried a `[VERIFIED against apps/desktop/main/engine.ts:77-79's resolvePackagedPythonPath]` comment (written during 03-01) returning `runtime/bin/python3.13` for darwin. Cross-checked against `.planning/phases/01-packaging-runtime-foundation/01-01-SUMMARY.md:50` (`python3 tools/provision-runtime.py` real run producing `engine/runtime/bin/python3.13`) — the citation is accurate and the path matches Phase 1's actual packaged output. `grep -n "ASSUMED\|VERIFIED" apps/desktop/main/launchagent-mac.ts` shows no `ASSUMED` marker. No code or comment change was required; no commit was made for this task since nothing changed.

**Task 2 — Wire darwin autostart install/detect/remediate into app startup:** In `apps/desktop/main/index.ts`:
- Added `import { detectState, install, resolveMacEnginePythonPath, LABEL, autostartRemediationDialog, LOGIN_ITEMS_SETTINGS_URL } from './launchagent-mac'`.
- Immediately after the closing brace of the engine attach/start block (the point where `directory` is already computed and the engine has successfully attached or started), added a `if (process.platform === 'darwin') { try { ... } catch (error) { console.warn('LaunchAgent install failed:', error) } }` block that:
  - Computes `entrypointPath`, `pythonPath` (via `resolveMacEnginePythonPath(directory)`), `logDir`/`stdoutLog`/`stderrLog`.
  - Calls `detectState(LABEL)` once.
  - On `'not-installed'`: calls `install({...})` — best-effort, errors caught by the outer try/catch and logged via `console.warn`, never thrown into the startup path.
  - On `'disabled'`: calls `dialog.showMessageBox(autostartRemediationDialog('disabled')).then(({response}) => { if (response === 0) void shell.openExternal(LOGIN_ITEMS_SETTINGS_URL) }).catch(...)`, mirroring the existing `showCrashReports` response-index wiring pattern.
  - Does nothing for `'ok'`/`'unknown'`.
- The entire block (including `detectState()` itself) sits inside one try/catch, satisfying the plan's prohibition that neither `install()` nor `detectState()` failures may throw uncaught into app startup.

## Verification

- `npm run typecheck` — exits 0 (both before and after reverting/reapplying the diff to confirm isolation).
- `npm test` (`node --test tests/*.test.mjs`) — 586 pass / 103 fail. Confirmed by temporarily reverting `apps/desktop/main/index.ts` to its pre-edit state and rerunning: **the same 103 tests fail identically** — all in `tests/update-rehearsal.test.mjs` and ACL-dependent tests (Windows drive-letter path assertions and real-ACL checks), pre-existing and unrelated to this plan's darwin-only, additive change. Diff was reapplied afterward (`git diff --stat` confirmed the same 22-line addition restored).
- `grep -n "process.platform === 'darwin'"` shows the new branch immediately following the engine attach/start block.
- `grep -c "launchctl load\|launchctl unload"` is 0.
- `git diff apps/desktop/main/index.ts` (task 2 commit) shows only additions — the existing `win32`/`configureTaskbar` line and surrounding control flow are unmodified.

## Deviations from Plan

**1. [Rule 3 - Blocking issue] Fast-forward merged `session/phase-3-launchd-autostart` (03-01's commits) into this worktree branch before starting.** The orchestrator's prompt asserted 03-01 was "already merged onto this branch," but this worktree's HEAD (`c9b6a95`) predated the 4 `03-01` commits (`8480d29`..`415e90f`), which lived only on `session/phase-3-launchd-autostart`. `launchagent-mac.ts` did not exist in this worktree at task start. Verified the merge-base of this worktree's HEAD and `session/phase-3-launchd-autostart` was exactly this worktree's HEAD (a clean fast-forward, no divergent commits), then ran `git merge --ff-only session/phase-3-launchd-autostart`. This brought in `apps/desktop/main/launchagent-mac.ts`, `.planning/phases/03-launchd-autostart/03-01-SUMMARY.md`, and the two 03-01 test files, with no conflicts and no rewriting of history.

**2. [Rule 3 - Blocking issue] Insertion point used structural position, not the plan's stale absolute line numbers.** The plan's action text anchored the edit to "line 1300/1301" (before `browserSession = session.fromPartition(...)`) and described the try block as ending right after `engine.start`. In the actual file, the try block encompasses the entire rest of startup (through line ~1620's `catch`), and the true "successful engine attach/start" point — the closing brace of `if (!await engine.attach(...))` — was at line 1328/1329, not 1300/1301 (concurrent phase work had shifted the file by ~28 lines since the plan was written). Inserted at the structurally correct point (immediately after that closing brace, before `session.fromPartition`), which satisfies the plan's stated intent and every acceptance criterion that doesn't hard-code an absolute line number.

**3. [Rule 1 - Bug avoidance] Added `.catch(() => {})` to the disabled-dialog `.then()` chain.** Not present in the plan's literal action text; added to avoid an unhandled promise rejection if `dialog.showMessageBox` rejects, matching an existing pattern already used elsewhere in `index.ts` (the `showUpdateFailure().then().catch()` chain).

**4. [Rule 1 - Bug avoidance] Corrected an incorrect requirement citation in a code comment before committing.** The first draft of the new darwin block's leading comment cited "DIST-01" for the no-settings-toggle decision; `DIST-01` is actually the unrelated code-signing/notarization requirement. Corrected to `BOOT-02` before committing (self-caught during review, not left in the shipped diff).

## Auth Gates

None.

## Human-Check Items (outstanding — escalate at end-of-phase UAT)

Per `workflow.human_verify_mode = end-of-phase` (no config override found), these two `<human-check>` items are carried forward for `/gsd:verify-work` to harvest into this phase's UAT.md — they were **not** attempted here as they require real macOS hardware and are explicitly non-automatable:

1. **BOOT-01 reboot/login survival (D-05):** On a real Mac, launch the built app once (darwin branch installs the LaunchAgent), then log out/in (or reboot). Expected: the engine process is running with no manual app launch — tray icon present, `launchctl print gui/$(id -u)/com.maurdekye.orgtree.boot-engine` reports `state = running` or `spawn scheduled`.
2. **BOOT-02 extended crash-loop false-positive check (Open Question 3, Assumption A3):** Install a deliberately crash-looping build as the LaunchAgent, leave it running for hours. Expected: `detectState()` does not report `'disabled'` while launchd is still throttle-retrying at its 10s `ThrottleInterval`; if BTM independently disables the item during the window, `detectState()` correctly classifies it as `'disabled'` on the next check.

These are the same two items named as prohibitions in the plan ("never skip... before `/gsd:verify-work` consolidates this phase's UAT.md") — they are being surfaced, not skipped.

## Threat Flags

None — this plan's new surface (the darwin block, the fixed `LOGIN_ITEMS_SETTINGS_URL` literal, the try/catch around `install()`/`detectState()`) is exactly what the plan's own `<threat_model>` (T-03-05, T-03-06, T-03-07) already covers; no new surface introduced beyond it.

## Self-Check: PASSED

- `apps/desktop/main/index.ts` — FOUND, contains `detectState`/`install`/`resolveMacEnginePythonPath`/`LABEL`/`autostartRemediationDialog`/`LOGIN_ITEMS_SETTINGS_URL` import and the new darwin block at line 1334.
- `apps/desktop/main/launchagent-mac.ts` — FOUND (brought in via the fast-forward merge), exports match exactly.
- Commit `eb2cea2` — FOUND in `git log --oneline`.
