# Codebase Concerns

**Analysis Date:** 2026-09-17

## macOS Port Blockers (Priority List)

This fork targets macOS. The upstream app is architecturally Windows-only in
its packaging, update, and OS-integration layers. Every item below must be
addressed (rewritten, branched, or explicitly stubbed) before a macOS build
is viable.

### 1. Packaging config has no macOS target at all
- File: `package.json` (`"build"` block)
- Current: `"win": { "target": "nsis" }` only. No `"mac"` key, no `"dmg"`,
  no `entitlements.plist`, no `entitlements.mac.inherit.plist`, no
  hardened-runtime config, no notarization credentials/config.
- Icon: `"icon": "apps/desktop/assets/orgtree-eye.ico"` — `.ico` only, no
  `.icns`. electron-builder requires a `.icns` for macOS app icon.
- `@electron/notarize` and `dmg-builder` are present only as *transitive*
  dependencies of `electron-builder` (see `package-lock.json`), never
  configured or invoked by this repo.
- npm scripts only cover Windows: `package:win`, `release:windows`,
  `verify:release` (Windows-log driven). There is no `package:mac` /
  `release:mac` equivalent.
- Fix approach: add a `"mac"` block (`target: dmg`/`zip`, `category`,
  hardened runtime, entitlements files), generate a `.icns`, add
  notarization (`afterSign` hook using `@electron/notarize` with an Apple ID
  / API key), and a `release:mac` script mirroring `release-windows.mjs`.

### 2. NSIS installer is the only installer, with deep custom NSIS scripting
- Files: `build/installer.nsh`, `build/installer-dev.nsh`,
  `build/update-fixture.nsi`
- These are hand-written NSIS (`.nsh`/`.nsi`) scripts wired into
  `electron-builder`'s `nsis.include` config. NSIS is Windows-only; there is
  no macOS equivalent notion of "include" scripts — DMG/pkg packaging is a
  completely different model (Info.plist, code signing, notarization,
  `dmg-license.txt` at most).
- Fix approach: none of this ports — write net-new mac packaging config;
  treat NSIS scripts as reference for *behavior* (elevation, upgrade
  handoff) that needs a mac-native equivalent, not code to port.

### 3. Installer elevation / relaunch is UAC + `elevate.exe`
- Files: `apps/desktop/main/index.ts` (`elevatorRunning`, `elevate.exe`
  sighting logic around line 622-762), `apps/desktop/main/updater.ts`
  (lines ~742-897, `resources/elevate.exe`, `InstallerSighting`,
  `InstallerProofSeams`)
- Detects whether the bundled `elevate.exe` (UAC broker) or the installer's
  own image is on screen, to distinguish "elevation prompt showing" from
  "installer running", purely via Windows process enumeration
  (`tasklist`/`wmic`-style sampling — see `sampleInstallerProcesses`).
- macOS has no UAC/elevate.exe equivalent; privilege escalation (if ever
  needed, e.g. for a `/Applications` write) uses `sudo`/`osascript
  "do shell script … with administrator privileges"` or, more idiomatically,
  is avoided entirely by installing to the user's own `~/Applications` or
  relying on DMG drag-install (no elevation at all).
- Fix approach: on macOS, drop this detection path entirely — DMG installs
  need no elevation step, so `elevatorRunning`/`InstallerSighting` should be
  Windows-only code paths, guarded and no-op on darwin.

### 4. Auto-update flow is electron-updater's NSIS differential-update path
- File: `apps/desktop/main/updater.ts` (715+ lines; heavy Windows-specific
  comments throughout, e.g. lines 374-1055)
- Relies on: NSIS's `/D=` install-directory override (with exact
  argument-ordering/quoting requirements verified against `makensis`),
  Windows registry reads for previous install scope/location
  (`InstallLocation REG_...` via `reg query`), `--updated /S --force-run`
  NSIS handoff flags, and `resources/elevate.exe` process sighting to know
  when a silent elevated handoff is in flight.
- None of this exists in `electron-updater`'s macOS path, which instead
  ships a `.zip` update artifact validated via Squirrel.Mac-style
  update semantics and code-signature checks (no elevation, no registry).
- Fix approach: this file needs a platform-forked update strategy —
  either maintain a parallel mac update path using `autoUpdater` events
  already abstracted by `electron-updater`, or gate the entire NSIS-specific
  branch behind `process.platform === 'win32'` and implement the mac branch
  separately. This is the single largest/highest-risk port item in the repo.

### 5. Background "boot engine" is a Windows Scheduled Task running as S4U
- Files: `tools/boot-engine-task.ps1`, `tools/register-boot-engine.ps1`,
  `tools/unregister-boot-engine.ps1`, `tools/boot-task-probe.ps1`
- PowerShell scripts that register a Scheduled Task (`Orgtree Background
  Engine`) with `<LogonType>S4U</LogonType>`, read/write a registry key
  `SOFTWARE\Orgtree\BootEngine`, and assert ACL/SID ownership via
  `System.Security.AccessControl` (SDDL strings, well-known SIDs like
  `S-1-5-18`, `S-1-5-32-544`).
- `apps/desktop/main/providerlogin.ts` (top-of-file comment) and
  `apps/desktop/main/engine.ts` (`Engine.attach`) both assume this boot-host
  engine may exist and prefer attaching to it over spawning a managed child.
- PowerShell (`.ps1`) does not exist as a native execution target on
  macOS (`pwsh` can be installed but isn't bundled, and Scheduled
  Tasks/registry/SDDL have no macOS analogue at all).
- macOS equivalent primitives: `launchd` (LaunchAgents/LaunchDaemons plists)
  for "start at boot/login", no registry (use a plist or app-support file),
  no Windows-style SID ACL model (use file permissions / keychain).
- Fix approach: this is a full rewrite, not a port — a macOS `launchd`
  agent plist + a shell/Python bootstrap script replacing all 4 `.ps1`
  files; `engine.ts`'s `Engine.attach` "prefer attached boot host" logic
  needs a macOS descriptor-file equivalent (the file-based descriptor/lock
  protocol itself, e.g. `engine-attach.json`, `.desktop-engine.lock`, looks
  platform-neutral and may be reusable).

### 6. Process-tree kill relies on `taskkill /T /F`
- Files: `apps/desktop/main/engine.ts` (`forceKillTree`, line ~451-455),
  `apps/desktop/main/providerlogin.ts` (`killTree`, line ~316-332)
- Already branches correctly: `if (process.platform !== 'win32') {
  process.kill(pid, 'SIGKILL') }`. This is one of the few places already
  macOS-aware — flagged here as a **verification item**, not a blocker: on
  POSIX, killing only the direct child does not kill descendants the way
  `taskkill /T` does. If any spawned CLI (provider login, engine launch)
  forks grandchildren, macOS will leak orphan processes where Windows would
  not. No process-group (`detached: true` + `process.kill(-pid)`) or job
  object equivalent is used on the POSIX branch.
- Fix approach: verify whether provider-CLI children ever fork further
  descendants; if so, spawn with `detached: true` and kill the process
  group (`process.kill(-pid, 'SIGKILL')`) on darwin/linux instead of a bare
  single-PID kill.

### 7. Windows-only native window chrome APIs
- File: `apps/desktop/main/taskbar.ts` — `window.setAppDetails(...)`
  (Electron API documented as Windows-only; no-op elsewhere per Electron
  docs, but relies on caller guarding).
- File: `apps/desktop/main/taskbar-attention.ts` — uses `flashFrame()`,
  which Electron implements on Windows and Linux but is a documented no-op
  on macOS. The macOS-idiomatic equivalent is `app.dock.bounce()`.
- Guarding: `apps/desktop/main/index.ts:188` already gates
  `configureTaskbar` behind `process.platform === 'win32'`, which is
  correct. `taskbar-attention.ts` itself has NO platform gate — it will run
  on macOS but `flashFrame(true)` silently does nothing, so the "taskbar
  pulse" UX feature (explicitly called out as a 2026-09-12 product
  decision in the file's own comment) is simply absent on macOS today.
- Fix approach: branch `TaskbarAttention.start()`/`stop()` to call
  `app.dock.bounce('informational')` / `app.dock.cancelBounce(id)` on
  darwin instead of `flashFrame`.

### 8. Tray icon assets and tray click behavior
- Files: `apps/desktop/assets/*.ico` (8 tray icon variants, `.ico` format
  only — no `.png`/Template `@2x` set for macOS menu-bar icons).
- `apps/desktop/main/traylist.ts` explicitly notes it renders a custom
  frameless popup because "a native Windows context menu cannot render an
  animated spinner or true columns" — this reasoning and the popup itself
  are platform-neutral (pure HTML popup), so likely portable, but the
  *icons* it uses are not: macOS menu-bar (`Tray`) icons should be template
  images (black + alpha, `Template` suffix) sized correctly for Retina, or
  they will render oversized/wrong-colored in the menu bar.
- Fix approach: add macOS-specific tray icon assets (`*Template.png` +
  `@2x`) and switch asset selection by platform where the tray icon is set.

### 9. Bundled Python runtime is Windows-embeddable-only
- File: `tools/provision-runtime.py`
- Hard `raise SystemExit(...)` unless `sys.platform == "win32"`; downloads
  `python-3.13.15-embed-amd64.zip` from python.org (Windows embeddable
  distribution) into `engine/runtime`, which is later packed into
  `extraResources` (`package.json` → `{"from": "engine", "to": "engine"}`).
- macOS has no "embeddable zip" distribution model; the standard approach
  is either bundling a `python-build-standalone` release (indygreg/
  python-build-standalone, portable, works on darwin arm64/x64) or
  requiring `pyenv`/Homebrew Python + a venv built at install time.
- `tools/stage-runtime.mjs` and `tools/runtime-layout.mjs` validate a
  runtime tree layout that is almost certainly Windows-shaped
  (`Lib/site-packages`, `._pth` file semantics are CPython-Windows-specific
  — POSIX python uses `lib/pythonX.Y/site-packages` and no `._pth`).
- `apps/desktop/main/engine.ts:185` reads `options.python` as an absolute
  path and just needs *some* valid interpreter — this consumption point
  is platform-neutral, but nothing populates `options.python` with a mac
  runtime today.
- Fix approach: write a `provision-runtime-mac.py` (or extend the existing
  script) using `python-build-standalone` mac builds, and adapt
  `runtime-layout.mjs`'s layout assertions to accept the POSIX layout.

### 10. `installer-relaunch.py` is a Windows GUI-subsystem hack
- File: `tools/installer-relaunch.py`
- Entire file exists to work around a Windows-only problem: NSIS's
  `StdUtils.ExecShellAsUser` cannot suppress a console window, so a
  GUI-subsystem Python (`pythonw.exe`) is used as the relaunch shim instead
  of PowerShell/WSH. This has zero macOS relevance — DMG-based installs
  have no "Setup relaunch" step at all (the app is just dragged/opened).
- Fix approach: delete/skip for the mac path — no equivalent needed.

### 11. Windows Job Object "guardian" process-tree ownership model
- File: `apps/desktop/main/engine.ts` — extensive comments about a
  "guardian" holding "the only handle" to descendant processes via what is
  described as a Windows Job Object (lines ~447-650: `guardianReleased`,
  `forceKillTree`, `stopAttachedForUpdate`), with a byte-range lock file
  (`.desktop-engine.lock`) used to detect guardian release.
- The byte-range-lock file protocol (`guardianReleased`) itself is
  POSIX-compatible (`fs` byte locks work cross-platform via `flock`-style
  semantics in Node), so this piece may need no change — but the *actual
  process-tree containment* (the Job Object that prevents orphaned
  descendants) has no POSIX equivalent already wired up. On macOS, nothing
  currently provides the same guarantee; only `SIGKILL` on the single known
  PID is issued (see item 6).
- Fix approach: needs its own design decision — likely a
  process-group-based guardian (spawn engine with `detached:true` in a new
  session, guardian kills the whole `-pgid`) rather than a literal port.

### 12. `.cmd`/`.bat` provider CLI wrapping assumes Windows shells
- File: `apps/desktop/main/providerlogin.ts:82` (`resolveArgv`) — wraps
  `.cmd`/`.bat` executables as `['cmd', '/c', exe, ...extra]`. This is
  already correctly gated (`if (process.platform === 'win32' && ...)`), so
  not a blocker — flagged only so the reviewer knows the POSIX branch
  (line ~126, `launchAntigravityTerminal`) is the one that needs real
  mac-side testing (`.sh`/binary CLIs, `$PATH` resolution via
  `~/.local/bin`).

### 13. Test suite assumes Windows-only tooling in several disruptive tests
- Files: `apps/desktop/renderer/tests/joblimit.ps1` (Windows Job Object
  launcher, invoked from `containment.test.ts`), `tests/*.ps1`
  (`boot-installer.test.ps1`, `boot-probe-preparation.test.ps1`,
  `paired-boot-preparation.test.ps1`, `probe-boot-paired-root-only.ps1`,
  `probe-boot-task-root-only.ps1`), `tools/test-installer-elevation.mjs`,
  `tools/test-installer-silent-elevation.mjs`,
  `tools/test-installer-update-mode.mjs`, `tools/test-installer-upgrade.mjs`,
  `tools/test-taskbar-native.mjs`, `tools/test-upgrade-close-boundary.mjs`.
- `apps/desktop/renderer/tests/containment.test.ts` already self-skips on
  non-Windows (`const skip = WIN ? false : 'joblimit.ps1 is a Windows Job
  Object launcher; not run on ${process.platform}'`) — a good pattern to
  replicate for the other PowerShell-backed tests, most of which do NOT yet
  appear to guard themselves (needs per-file verification before porting).
- Fix approach: audit each installer/taskbar-native test for a platform
  guard; where absent, add one before running the suite on a mac CI runner.
  There is currently only one CI workflow, `.github/workflows/startup-budget.yml`
  — no macOS runner/matrix entry exists yet.

### 14. `C:\` and `SystemRoot` hardcoded paths in tests
- Files: `apps/desktop/renderer/tests/containment.test.ts:38`
  (`path.join(process.env.SystemRoot ?? 'C:\\Windows', ...)`),
  `apps/desktop/renderer/tests/eventdecode.test.ts:31,36` (test fixtures
  using literal `C:\already-visible text` / `C:\secret-key` strings — these
  are just log-redaction test fixtures, not runtime logic, so lower
  priority, but confirm they don't accidentally assert Windows-only
  redaction regexes that wouldn't match a macOS `/Users/...` path).

## Tech Debt

**Windows registry as a source of truth for install location:**
- Files: `apps/desktop/main/index.ts` (`readRegisteredInstallLocation`,
  `readInstallScope`, lines ~505-650), `apps/desktop/main/updater.ts`
  (multiple registry reads for previous install scope).
- Impact: the entire "detect per-user vs per-machine install, and where"
  logic has no macOS analogue (there is no registry; app location is just
  `/Applications/Orgtree.app` or wherever the user dragged it). Any
  consumer of `canInstallUnattended()`/`installedForAllUsers` needs a
  macOS-specific implementation or an explicit "always per-user, no
  unattended install" default.

## Fragile Areas

**`updater.ts` (715+ lines):**
- Files: `apps/desktop/main/updater.ts`
- Why fragile: the file's own comments describe multiple previously-shipped
  bugs (e.g. the `/D=` argument-ordering/quoting parser mismatch verified
  against real `makensis`, tracked via `tests/disruptive/nsis-destination.test.mjs`)
  and encodes very specific, easily-broken assumptions about NSIS internals
  and `elevate.exe` process-sighting timing.
- Safe modification: any change here needs the disruptive test suite
  (`npm run test:disruptive`) run on real Windows, since the logic is
  fundamentally OS/installer-version-coupled, not something unit tests on
  macOS can meaningfully validate.

**`engine.ts` guardian/attach protocol:**
- Files: `apps/desktop/main/engine.ts`
- Why fragile: relies on a Windows Job Object (via an external "guardian"
  process not shown in this file) plus a byte-range lock file as a
  cross-process death/release signal. The POSIX fallback path
  (`process.kill(pid, 'SIGKILL')`) does not provide the same tree-teardown
  guarantee, so behavior under process failure will differ subtly on
  macOS vs. Windows until a POSIX guardian equivalent exists.

## Test Coverage Gaps

**No macOS-specific tests exist yet:**
- What's not tested: tray icon rendering (Template image correctness),
  dock bounce behavior, `launchd` boot-engine registration/lifecycle,
  DMG install/uninstall, code-signing/notarization gate, mac Python runtime
  provisioning.
- Files: none exist under `tools/test-*` or `tests/` for any of the above;
  compare to the extensive Windows equivalents (`test-installer-elevation.mjs`,
  `test-taskbar-native.mjs`, `boot-installer.test.ps1`, etc.).
- Risk: every item in the "macOS Port Blockers" section above currently
  ships with zero regression coverage on the target platform.
- Priority: High — should be added alongside each blocker's fix, not
  after the fact.

---

*Concerns audit: 2026-09-17*
