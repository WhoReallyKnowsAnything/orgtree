---
phase: 03-launchd-autostart
plan: 01
subsystem: infra
tags: [launchd, launchctl, macos, plist, autostart, boot-engine]
requires: []
provides: [launchagent-mac.ts lifecycle module (buildPlist, install, uninstall, detectState, autostartRemediationDialog)]
affects: [apps/desktop/main/index.ts (Plan 02 wiring)]
tech-stack:
  added: []
  patterns: [plist template + plutil -lint gate, launchctl bootstrap/bootout only (no load/unload), exit-code-first three-layer state detection]
key-files:
  created:
    - apps/desktop/main/launchagent-mac.ts
    - tests/disruptive/launchagent-install.test.mjs
    - tests/launchagent-detect.test.mjs
  modified: []
decisions:
  - "resolveMacEnginePythonPath corrected from the plan's [ASSUMED] unversioned python3 to the VERIFIED runtime/bin/python3.13 subpath, matching apps/desktop/main/engine.ts's already-shipped resolvePackagedPythonPath (Phase 1 packaging had, in fact, already landed in this worktree contrary to the plan's premise)"
  - "sfltool dumpbtm BTM-disallowed detection parses blank-line-separated blocks and matches a block containing both the label and a Disposition: ... disallowed pattern, since dumpbtm's output format is undocumented and this is enrichment-only, never the sole signal"
metrics:
  duration: ~45m
  completed: 2026-09-23
status: complete
actuals:
  tokens: 4498
  tasks: 3
  commits: 3
---

# Phase 3 Plan 01: LaunchAgent Lifecycle (build/install/uninstall/detect/remediate) Summary

Darwin-only LaunchAgent lifecycle module replacing the Windows Scheduled Task boot-engine
lifecycle: plist template generation with `plutil -lint` validation, install/uninstall via
`launchctl bootstrap`/`bootout` only, three-layer exit-code-first state detection
(not-installed/disabled/ok), and remediation dialog content mirroring the existing
`crashReportDialog` convention.

## What Was Built

**`apps/desktop/main/launchagent-mac.ts`** — zero-Electron-import, standalone-bundleable module:

- `LABEL` — `com.maurdekye.orgtree.boot-engine`, namespaced under the app's own appId.
- `resolveMacEnginePythonPath(directory)` — the one isolated place the packaged Python
  interpreter subpath lives.
- `xmlEscape(value)` — escapes `&`, `<`, `>`, `"`, `'` for every value interpolated into the
  plist XML.
- `buildPlist(options)` — builds the plist string with `KeepAlive` always
  `{SuccessfulExit: false}`, matching `docs/engine-contract.md:16`'s shutdown contract.
- `plistPath(label)` — `~/Library/LaunchAgents/<label>.plist`.
- `install(options)` — writes the plist to the canonical path, runs `plutil -lint` (throws on
  nonzero, which is the lint gate), then `launchctl bootstrap gui/<uid> <path>`.
- `uninstall(label, uid)` — `launchctl bootout gui/<uid>/<label>` (swallowing an
  already-removed error), then removes the plist file.
- `AutostartState` + `detectState(label, deps)` — three-layer classification:
  `launchctl print` exit code (primary), `launchctl print-disabled` text match (secondary,
  launchd-level re-enableable disable), `sfltool dumpbtm` BTM `Disposition: disallowed` match
  (tertiary, best-effort — the actual "no programmatic re-enable" case). Never throws.
- `LOGIN_ITEMS_SETTINGS_URL` + `autostartRemediationDialog(state)` — dialog content for
  `'not-installed'` and `'disabled'`, shaped identically to `crashReportDialog`.

**`tests/disruptive/launchagent-install.test.mjs`** — gated (via `requireDisruptiveOptIn`),
proves a real `bootstrap → print → bootout → print` round-trip against a throwaway label
(`com.maurdekye.orgtree.disruptivetest.<uuid>`) using `/usr/bin/true` as the spawned program so
it exits 0 immediately and is never respawned. Cleanup in `test.after` removes the plist, the
esbuild temp dir, and force-runs `bootout` even if an assertion fails mid-test. Verified live on
this machine (Darwin 27.2, arm64): no stray LaunchAgent remained after the run.

**`tests/launchagent-detect.test.mjs`** — unit tests for `detectState` and
`autostartRemediationDialog` against a mocked `execFileSyncImpl`; no real `launchctl`/`sfltool`
call in this file.

## Deviations from Plan

### Auto-fixed Issues

**1. [Rule 1 - Bug] Corrected `resolveMacEnginePythonPath`'s subpath from an unversioned guess to the already-verified Phase 1 output**
- **Found during:** Task 1 implementation, while reading `apps/desktop/main/engine.ts` for the
  existing execFileSync convention.
- **Issue:** The plan's `<action>` block, written against Open Question 1's `[ASSUMED]` framing
  ("Phase 1 packaging... have not executed in this worktree"), specified
  `path.join(directory, 'runtime', 'bin', 'python3')`. But Phase 1 packaging has, in fact,
  already landed in this worktree: `apps/desktop/main/engine.ts:77-79` exports
  `resolvePackagedPythonPath`, which for darwin already resolves to
  `path.join(directory, 'runtime', 'bin', 'python3.13')` — confirmed by a real fixture-backed
  test (`tests/resolve-packaged-python-path.test.mjs`). Shipping the plan's unversioned guess
  would have pointed the LaunchAgent at a nonexistent binary in production.
- **Fix:** `resolveMacEnginePythonPath` returns `path.join(directory, 'runtime', 'bin',
  'python3.13')`, with its docstring citing `engine.ts:77-79` as the verified source instead of
  the plan's `[ASSUMED]` citation. Kept as its own function in `launchagent-mac.ts` (not an
  import from `engine.ts`) so this module stays a standalone, Electron-free bundle — the
  isolation the plan's must_have asked for is preserved, only the value inside it changed.
- **Files modified:** `apps/desktop/main/launchagent-mac.ts`.
- **Commit:** 8480d29.

Or otherwise: plan executed as written for all other behavior.

## Self-Check: PASSED

- FOUND: apps/desktop/main/launchagent-mac.ts
- FOUND: tests/disruptive/launchagent-install.test.mjs
- FOUND: tests/launchagent-detect.test.mjs
- FOUND commit 8480d29 (Task 1)
- FOUND commit 880d299 (Task 2)
- FOUND commit cb46da1 (Task 3)
- `npm run typecheck` exits 0
- `node --test tests/launchagent-detect.test.mjs` — 10/10 pass
- `ORGTREE_DISRUPTIVE_PROBES=1 node --test tests/disruptive/launchagent-install.test.mjs` — 2/2 pass, real bootstrap/bootout round-trip verified, no stray LaunchAgent left
- `node --test tests/disruptive-policy.test.mjs` — 6/6 pass (repo-wide gate policy unaffected)
