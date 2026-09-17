# Phase 3: Launchd Autostart - Pattern Map

**Mapped:** 2026-09-17
**Files analyzed:** 6 (new)
**Analogs found:** 6 / 6

## File Classification

| New File | Role | Data Flow | Closest Analog | Match Quality |
|----------|------|-----------|-----------------|----------------|
| `apps/desktop/main/launchagent.ts` | service (OS-integration lifecycle: generate plist, install/uninstall/detect) | request-response (shell out to `launchctl`/`plutil`, parse result) | `tools/boot-engine-task.ts` — actually PowerShell (`tools/boot-engine-task.ps1`); best TS analog is `apps/desktop/main/providerlogin.ts` (child_process invocation conventions) + `apps/desktop/main/engine.ts` (lifecycle state machine shape) | role-match (cross-language for the XML/plist half, exact-match for the child_process half) |
| `apps/desktop/main/index.ts` (modified — wire install/detect calls into startup) | controller (Electron main entrypoint) | event-driven (app lifecycle hooks) | existing `index.ts` itself — extend in place | exact (same file) |
| `tests/launchagent-detect.test.mjs` | test (unit, mocked `launchctl` output) | request-response | `tests/process-failure.test.mjs` | exact |
| `tests/disruptive/launchagent-install.test.mjs` | test (disruptive/integration, real `launchctl bootstrap`/`bootout`) | request-response | `tests/boot-installer.test.ps1` (Windows disruptive install test) — no existing `tests/disruptive/*.mjs` file yet, so `tests/process-failure.test.mjs`'s esbuild-bundle pattern is the closest *mjs* analog for structure | role-match |
| plist template (embedded string constant, no standalone file — see D-03/research) | config/template | transform (string interpolation → validated via `plutil -lint`) | `tools/boot-engine-task.ps1`'s `New-BootTaskXml` (XML DOM build + validate) | role-match (same "hand-build small XML, validate before use" pattern, different language) |
| `docs/engine-contract.md` (read-only reference, not modified) | — | — | n/a | n/a (constraint source for `KeepAlive.SuccessfulExit`) |

## Pattern Assignments

### `apps/desktop/main/launchagent.ts` (service, request-response)

**Analog 1 — child_process invocation convention:** `apps/desktop/main/providerlogin.ts`

**Imports pattern** (providerlogin.ts:1):
```typescript
import { spawn, execFileSync, type ChildProcessWithoutNullStreams } from 'node:child_process'
```
Use `execFileSync` for `launchctl print`, `plutil -lint`, `launchctl bootstrap`/`bootout` — all synchronous, short-lived, argv-array calls (no shell string interpolation). This matches the project convention already established for shelling out from the Electron main process; no new dependency needed (confirmed in RESEARCH.md "Standard Stack").

**Platform guard pattern** (seen repeatedly — policy.ts:53, engine.ts:453, providerlogin.ts:82):
```typescript
if (process.platform !== 'win32') { /* mac/posix branch */ }
```
Invert for the new module: `if (process.platform === 'darwin') { ... }` guards around all launchd calls — no existing `darwin`-branch code exists yet in the codebase (confirmed via search), so this module is the first to introduce it. Follow the same inline-guard style rather than a new abstraction layer.

**Analog 2 — lifecycle state machine shape:** `apps/desktop/main/engine.ts` (`class Engine`, `§+Engine @L76-675`)
Engine.ts models install/attach/stop/restart as discrete async methods on a class (`attach`, `start`, `stop`, `restart`, `verifyAttached`) rather than one big function — mirror this shape for `launchagent.ts`: separate `install()`, `uninstall()`, `detectState()` functions/methods, each independently testable, matching CONTEXT.md's explicit instruction ("keep install/uninstall/detect as discrete, independently testable operations", quoting the Windows `Invoke-BootLifecycle` precedent).

**Analog 3 — XML/plist template build + validate:** `tools/boot-engine-task.ps1:130` `New-BootTaskXml($Record)`
```powershell
function New-BootTaskXml($Record) {
    $paths = Get-BootPaths $Record.InstallDir
    # Serialize Unicode/XML escaping. Command is an unquoted path, not shell
    # text; the separate argument is the quoted service-host script path.
    $doc = Read-BootXml @'
<Task version="1.2" ...><Actions>...
```
Port the *pattern*, not the code: build the plist as a literal XML/template-string constant with placeholders substituted by trusted internal values only (install dir, executable path — never untrusted user input), then validate the *result* before use. RESEARCH.md's Alternatives Considered table explicitly endorses this over pulling in an npm `plist` package, citing this exact PowerShell function as the project's own precedent for "build simple XML, validate it."

Plist skeleton to build (from 03-RESEARCH.md lines ~170-195):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.maurdekye.orgtree.boot-engine</string>
    <key>ProgramArguments</key>
    <array><string>/path/to/engine</string></array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key>
        <false/>
    </dict>
    <key>WorkingDirectory</key>
    <string>/path/to/engine</string>
    <key>StandardOutPath</key>
    <string>/path/to/logs/boot-engine.out.log</string>
    <key>StandardErrorPath</key>
    <string>/path/to/logs/boot-engine.err.log</string>
</dict>
</plist>
```
`Label` = `com.maurdekye.orgtree` (from `package.json:54` `"appId": "com.maurdekye.orgtree"`) + a namespaced suffix, replacing the Windows task name `'Orgtree Background Engine'` (`tools/boot-engine-task.ps1:3` `$script:BootTaskName`).
`KeepAlive.SuccessfulExit = false` is **required**, not optional — mirrors the engine's own shutdown contract at `docs/engine-contract.md:16`: "Requested shutdown exits 0 ... engine failure exits nonzero." Bare `KeepAlive: true` would fight this contract (RESEARCH.md Pitfall 3).

**Error handling pattern** (register-boot-engine.ps1, unregister-boot-engine.ps1 — the try/catch shape to mirror conceptually, not literally):
```powershell
try {
    . "$PSScriptRoot\boot-engine-task.ps1"
    Invoke-BootLifecycle -Action $Action -InstallDir $InstallDir -OperatorSid $OperatorSid -InstallMode $InstallMode
    Write-Output "Boot engine $Action completed."
} catch { [Console]::Error.WriteLine($_.Exception.Message); exit 1 }
```
TS equivalent: each discrete operation (`install`/`uninstall`/`detectState`) should throw/return a typed result on failure rather than let `execFileSync` exceptions propagate uncaught — `engine.ts`'s async methods (e.g. `finish(error?: Error)` at L206-234) show the project's convention of capturing an `Error | undefined` and centralizing the failure path.

### `apps/desktop/main/index.ts` (modified)

**Analog:** itself. Wire `launchagent.install()`/`detectState()` calls into the existing startup sequence the same way `configureTaskbar` is gated on `process.platform === 'win32'` (index.ts:188) — add a symmetric `process.platform === 'darwin'` branch calling into `launchagent.ts`. Do not restructure `index.ts`'s existing control flow beyond adding this branch.

### `tests/launchagent-detect.test.mjs` (unit test)

**Analog:** `tests/process-failure.test.mjs` (full pattern, lines 1-54 read in full)

**Bundling pattern** (process-failure.test.mjs:15-21):
```javascript
import { build } from 'esbuild'
import { createRequire } from 'node:module'

const repo = path.resolve(import.meta.dirname, '..')
const root = fs.mkdtempSync(path.join(os.tmpdir(), 'orgtree-process-failure-'))
const outfile = path.join(root, 'process-failure.cjs')
await build({ entryPoints: [path.join(repo, 'apps/desktop/main/process-failure.ts')], outfile, bundle: true, format: 'cjs', platform: 'node' })
const { attachRendererFailureHandlers, ... } = createRequire(import.meta.url)(outfile)
```
Apply identically to `launchagent.ts`: bundle via esbuild to a temp `.cjs`, `createRequire` it, import the exported functions. This is the established project convention for testing main-process TS modules under `node:test` without a full Electron harness.

**Dependency-injection / mock pattern** (process-failure.test.mjs:38-54, `harness(options)`):
```javascript
function harness(options = {}) {
  const records = [], announced = [], gaveUp = []
  let reloads = 0, clock = options.start ?? 1_000_000
  const contents = emitter()
  const hooks = { record: (stage, detail) => records.push({ stage, detail }), ... , now: () => clock }
  attachRendererFailureHandlers(contents, hooks, new RecoveryBudget(options.limit, options.windowMs))
  return { contents, records, ... }
}
```
Apply the same shape to `launchagent.ts`'s `detectState()`: design it to accept an injectable `execFileSync`-like function (or a `launchctlPrint: () => string` hook) so tests can feed mocked `launchctl print` output strings and assert the returned classification (not-installed / launchd-disabled / ok) without touching a real launchd. Matches BOOT-02's test-map row: "Detection logic correctly classifies ... given mocked `launchctl` output" (03-RESEARCH.md line 393).

**Cleanup pattern** (process-failure.test.mjs pairs with registry-labels.test.mjs:78):
```javascript
test.after(() => { fs.rmSync(dir, { recursive: true, force: true }) })
```
Always clean up the temp esbuild output dir.

### `tests/disruptive/launchagent-install.test.mjs` (disruptive/integration test)

**No direct .mjs analog exists** — `tests/disruptive/` is currently empty (0 files). Closest analog is the *intent* of `tests/boot-installer.test.ps1` (Windows real-registration disruptive test) combined with the esbuild-bundle mechanics of `tests/process-failure.test.mjs`. This satisfies D-05 ("the plan must include a real verification step — actually install/uninstall a test LaunchAgent and observe `launchctl print` / process state"). Structure: install a real but uniquely-named/throwaway LaunchAgent plist, `launchctl bootstrap`, `launchctl print` to confirm state, `launchctl bootout`, `launchctl print` again to confirm removal — never touch the app's real production Label.

## Shared Patterns

### child_process invocation
**Source:** `apps/desktop/main/providerlogin.ts:1, :82` (`execFileSync`, `resolveArgv`)
**Apply to:** `launchagent.ts`'s calls to `launchctl`, `plutil`
```typescript
import { execFileSync } from 'node:child_process'
// argv array form — never shell string interpolation
execFileSync('launchctl', ['bootstrap', `gui/${uid}`, plistPath])
```

### Platform guard
**Source:** `apps/desktop/main/policy.ts:53`, `engine.ts:453`, `index.ts:188`
**Apply to:** every entry point into `launchagent.ts` from `index.ts`
```typescript
if (process.platform === 'darwin') { /* launchagent install/detect */ }
```

### Build-simple-XML-and-validate (no library)
**Source:** `tools/boot-engine-task.ps1:130` `New-BootTaskXml`
**Apply to:** plist template generation in `launchagent.ts`
Hand-build the plist as a template string with trusted-value interpolation only; validate via `plutil -lint <path>` (execFileSync) before every `bootstrap` call, per D-03.

### esbuild-bundle-then-node:test unit testing
**Source:** `tests/process-failure.test.mjs:15-21`
**Apply to:** `tests/launchagent-detect.test.mjs`
Bundle the main-process TS module to CJS via esbuild into a tmp dir, `createRequire` it, test exported pure functions directly — no Electron runtime needed for unit-level logic.

## No Analog Found

| File | Role | Data Flow | Reason |
|------|------|-----------|--------|
| `tests/disruptive/launchagent-install.test.mjs` | test | request-response | `tests/disruptive/` directory exists but is empty (0 files) — no prior disruptive `.mjs` test to copy structure from directly; closest is the Windows `.ps1` disruptive test's *intent*, described above. |

## Metadata

**Analog search scope:** `apps/desktop/main/`, `tools/`, `tests/`, `package.json`, `docs/engine-contract.md`
**Files scanned:** ~12 (providerlogin.ts, engine.ts, process-failure.ts, policy.ts, index.ts, installer-upgrade.ts, harnesses.ts, boot-engine-task.ps1, register-boot-engine.ps1, unregister-boot-engine.ps1, boot-task-probe.ps1, process-failure.test.mjs, registry-labels.test.mjs)
**Pattern extraction date:** 2026-09-17
