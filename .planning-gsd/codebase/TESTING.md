# Testing Patterns

**Analysis Date:** 2026-09-17

## Test Framework

**Two independent suites — TS (Node) and Python — plus a separate renderer suite:**

| Suite | Runner | Config | Command |
|---|---|---|---|
| Main-process/build/installer TS | Node's built-in `node:test` | none (no jest/vitest/mocha) | `npm test` → `node --test tests/*.test.mjs` |
| Disruptive (opt-in, disturbs the OS) | `node:test` | `tests/disruptive/gate.mjs` opt-in gate | `npm run test:disruptive` → `node --test tests/disruptive/*.test.mjs` |
| Renderer (React/TSX) | `node:test` + jsdom, bundled per-file with esbuild | `apps/desktop/renderer/tests/run.mjs` (custom runner, no config file) | `npm run test:renderer` |
| Backend engine (Python) | `unittest` (class-based `TestCase`) | no `pytest.ini`/`conftest.py` found — plain unittest discovery | run via `unittest`/whatever the engine's own tooling invokes (`tools/run-python-verification.py`) |

**Assertion Library:** Node's `node:assert/strict` for all TS suites; Python's `unittest.TestCase` assertions for the backend.

**Run Commands:**
```bash
npm test                    # TS main/build/installer tests — node --test tests/*.test.mjs
npm run test:disruptive     # opt-in disruptive probes (installers, elevation, console windows)
npm run test:renderer       # React/TSX renderer suite via jsdom + esbuild bundling
npm run test:electron       # Electron-specific harness (tools/test-electron.mjs)
npm run test:known-failures # show the recorded baseline of known-failing tests
npm run test:compare        # compare current run against recorded baseline
npm run test:baseline:record # record a new known-failures baseline
```

## Test File Organization

**Location:**
- TS main/build/installer tests: flat in `tests/*.test.mjs` (only non-recursive glob reach — this is deliberate, see Disruptive Tests below)
- Disruptive/dangerous TS probes: `tests/disruptive/*.test.mjs`, plus shared helpers `tests/disruptive/gate.mjs`
- Acceptance/E2E harness: `tests/acceptance/` (`harness.test.mjs`, `run.mjs`, `isolation.mjs`, and per-area `run_*.mjs` files: `run_artifacts.mjs`, `run_connections.mjs`, `run_history.mjs`, `run_lifecycle.mjs`, `run_maintenance.mjs`, `run_management.mjs`, `run_relaunch.mjs`, `run_unstick.mjs`)
- Shared fixtures for TS tests: `tests/fixtures/` (`benchmark.mjs`, `console-window-lock.mjs`, `portal.mjs`)
- Renderer (React) tests: co-located under `apps/desktop/renderer/tests/*.test.tsx`, one file per component/behavior, named after the component/scenario in lowercase (`agentstray.test.tsx`, `zoombutton.test.tsx`)
- Renderer visual/DOM probes (Python, not pytest): `apps/desktop/renderer/tests/*_probe.py` — ad hoc DOM/layout inspection scripts, not part of the automated `npm test` run
- Backend Python tests: flat in `tests/test_*.py` at repo root (large suite — 300+ files covering accounts, agents, antigravity/claude/codex provider parity, engine lifecycle)

**Naming:**
- `*.test.mjs` / `*.test.tsx` for TS; `test_*.py` for Python `unittest`
- Python test class names describe the subsystem under test (`RegistryTests` in `test_account_registry.py`)

**Structure:**
```
tests/
  *.test.mjs                 # main-process/build/installer, node --test default glob
  test_*.py                  # backend python, unittest, 300+ files
  disruptive/*.test.mjs      # gated, OS-disturbing probes (installers, elevation, console windows)
  acceptance/*.mjs           # E2E acceptance harness + per-feature runners
  fixtures/*.mjs             # shared TS test fixtures
apps/desktop/renderer/
  tests/*.test.tsx           # React component tests (jsdom + esbuild bundling)
  tests/harness.ts           # shared rig: jsdom setup, FakeServer, fetch stub, mock timers
  tests/run.mjs              # custom bundling test runner (no jest/vitest)
  tests/*_probe.py           # manual visual/DOM inspection scripts, not in CI test run
```

## Test Structure

**TS suite pattern (`node:test`):**
```javascript
import test from 'node:test'
import assert from 'node:assert/strict'
// ... build/import the module under test via esbuild, since main-process
// code is TypeScript with extensionless imports and this repo has no
// jest/ts-jest transform layer:
async function load(name) {
  const out = path.join(temp, name + '.cjs')
  await build({ entryPoints: [`apps/desktop/main/${name}.ts`], outfile: out, bundle: true, platform: 'node', format: 'cjs' })
  return req(out)
}
const channel = await load('build-channel')

test('dev version names the exact source commit and tree state', () => {
  assert.equal(devVersion('2.0.9', 'ab12cd34ef99', false), '2.0.9-dev.gab12cd34ef')
})
```
(`tests/dev-install.test.mjs`)

**Python suite pattern (`unittest`):**
```python
class RegistryTests(unittest.TestCase):
    @classmethod
    def setUpClass(cls):
        cls.root = tempfile.mkdtemp(prefix="orgtree-registry-")
        os.environ["ORGTREE_DATA"] = cls.root
        from engine.backend.orgtree import registry, store
        cls.registry = registry

    def setUp(self):
        path = self.registry.registry_path()
        if os.path.exists(path):
            os.unlink(path)
```
(`tests/test_account_registry.py`) — module docstring names the design doc the tests verify (`"Design: v2-accounts-design.md draft 6"`), and calls out *why* a negative case exists next to its positive counterpart.

**Patterns:**
- Setup: temp directories via `tempfile.mkdtemp`/`fs.mkdtempSync`, never touching real user data directories
- Platform-conditional skip: `{ skip: process.platform !== 'win32' ? 'Windows only' : false }` passed as the third arg to `test()` in TS (`tests/installer-upgrade.test.mjs`); `raise unittest.SkipTest("NTFS ACLs are Windows-only")` in Python (`tests/test_service_host.py`)
- Assertion style favors regex matching on process output for CLI/installer tests: `assert.doesNotMatch(relaunch, /winmgmts|Win32_Process|ActiveXObject|WScript|win32com/)`

## Mocking

**Framework:** No mocking library. Node's built-in `node:test` `mock` module (`import { mock } from 'node:test'`) plus hand-written fakes.

**Patterns:**
```typescript
// apps/desktop/renderer/tests/harness.ts — the shared rig every renderer suite mounts on
import { JSDOM } from 'jsdom'
import { mock } from 'node:test'
// provides: jsdom window/document/localStorage on globalThis, a FakeServer
// simulating the backend's own view of a conversation, a fetch stub with
// programmable latency/out-of-order delivery/failure injection, and
// deterministic time via node:test's mock timers (`mock.timers`)
```
Renderer tests must import `harness.ts` **first**, before any `../src/*` import — `api.ts` reads `location.pathname` and `desk.tsx` reads `localStorage` at module scope, so the DOM has to exist before those module bodies run. This import-order dependency is called out with an explicit `⚠` warning comment; violating it silently breaks the test rather than throwing clearly.

**What to Mock:**
- The backend HTTP surface (via the `fetch` stub in `harness.ts`) and wall-clock time (via `node:test` mock timers) in renderer tests.
- The compiled Electron-builder NSIS templates for installer tests, via monkey-patching `NsisTarget.prototype.executeMakensis` to intercept script generation without ever executing a real installer (`tests/disruptive/compile-boot-installer.cjs`).

**What NOT to Mock:**
- React's own subscription/render path — `harness.ts` explicitly exercises `mountConvoView()` as "a real React subscriber, i.e. the actual code path a desk uses, so subscription-gated liveness is exercised rather than faked."
- `jsdom`'s `pretendToBeVisual` is deliberately left OFF — it starts a real 60Hz rAF loop on the real clock that would hold the Node event loop open forever.

## Fixtures and Factories

**Test Data:**
- TS: helper functions per test file build minimal fixture objects inline (no shared factory library); shared reusable fixtures live in `tests/fixtures/*.mjs` (`benchmark.mjs`, `console-window-lock.mjs`, `portal.mjs`)
- Python: `_mk(...)` helper methods on the test class build minimal domain objects inline (`RegistryTests._mk` in `test_account_registry.py`)

**Location:**
- TS shared fixtures: `tests/fixtures/`
- Python: no shared fixture module found — each test file defines its own helpers

## Coverage

**Requirements:** None enforced by tooling — no coverage config, no CI coverage gate found in the explored files.

**Known-failures baseline (non-standard but load-bearing):**
- `npm run test:known-failures` / `test:compare` / `test:baseline:record` (backed by `tools/test-baseline.mjs`) track a recorded baseline of tests known to fail on a given platform, rather than a coverage percentage — this is the project's substitute for a coverage gate and is specifically useful for tracking macOS-port gaps without blocking the whole suite.

## Test Types

**Unit Tests:** `tests/*.test.mjs` (TS main-process modules bundled individually with esbuild), `tests/test_*.py` (Python backend logic, `unittest.TestCase`), `apps/desktop/renderer/tests/*.test.tsx` (React components in jsdom).

**Integration/Acceptance Tests:** `tests/acceptance/` — a harness (`harness.test.mjs`, `harness.ts`) plus per-feature runner scripts (`run_lifecycle.mjs`, `run_management.mjs`, `run_relaunch.mjs`, `run_unstick.mjs`, etc.) that appear to drive multi-component flows (organization lifecycle, connections, history) rather than isolated units.

**Disruptive/Destructive Tests (Windows-specific — critical for the macOS port):**
- `tests/disruptive/` holds probes that intentionally disturb the machine: open a console window, show a UAC elevation prompt, or run a compiled NSIS installer. Reachable only via `npm run test:disruptive` with `ORGTREE_DISRUPTIVE_PROBES=1` set (`tests/disruptive/gate.mjs`) — two deliberately redundant barriers (the file glob doesn't recurse into `disruptive/`, AND the gate function must return true).
- `tests/installer-elevation.test.mjs` and `tests/installer-upgrade.test.mjs`: many individual `test()` calls guarded with `{ skip: process.platform !== 'win32' ? 'Windows only' : false }` — these exercise NSIS installer scripts, `Start-Process -Verb RunAs`-style elevation, PowerShell (`WindowsPowerShell/v1.0`), and Windows registry/WMI avoidance checks (`assert.doesNotMatch(..., /winmgmts|Win32_Process|ActiveXObject|WScript|win32com/)`). These are Windows-only by construction (NSIS/PowerShell/registry don't exist on macOS) and **will not run at all on a macOS CI box** — they report as skipped, not failing, which can mask a real regression if nobody notices they never ran.
- `tests/test_service_host.py`: three tests explicitly `raise unittest.SkipTest("NTFS ACLs are Windows-only")` / `"NTFS ownership is Windows-only"` — these guard the Windows service-host's file-permission model and have no macOS equivalent test yet. **When porting the equivalent POSIX permission model to macOS, write new companion tests here rather than trying to make these pass under Wine/emulation.**
- `tests/disruptive/nsis-destination.test.mjs`, `tests/disruptive/installer-log.test.mjs`, `tests/disruptive/console-lifetime.test.mjs`, `tests/disruptive/lifetime-controls.test.mjs`: all assume an NSIS-built `.exe` installer and Windows console/window semantics — none of this has a macOS analog yet (no `.pkg`/`.dmg`/notarization test scaffolding exists in the repo as of this analysis).
- `tests/disruptive/compile-boot-installer.cjs`, `boot-installer.test.ps1`: the latter is a literal PowerShell script — cannot execute on macOS at all, would need a `.sh`/bash equivalent or removal once the installer story moves off NSIS.

**Taskbar/tray-native Windows probes:**
- No dedicated `taskbar`-named test file was found in `tests/`, but tray/menu logic (`refreshTrayEngineMenu`, `trayEngineState` in `apps/desktop/main/engine.ts`) and multi-provider tray icon assets (`apps/desktop/assets/orgtree-eye-tray-*.ico`) are Windows-icon-format (`.ico`) specific. `tests/icon-assets.test.mjs` matched the Windows/win32 search and should be checked for `.ico`-only assumptions when adding macOS `.icns`/menu-bar equivalents.

**E2E Tests:** No separate E2E framework (no Playwright/Spectron/WebdriverIO found) — `tests/acceptance/` plus `test:electron` (`tools/test-electron.mjs`) fill this role using the same `node:test`-based approach as the rest of the suite.

## Common Patterns

**Async Testing:**
```javascript
test('...', async () => {
  const result = await someAsyncOperation()
  assert.equal(result, expected)
})
```
Standard `async () => {}` test bodies throughout; no custom async helpers beyond Node's native `node:test` support.

**Platform-Conditional Testing (the pattern to reuse when adding macOS-only tests):**
```javascript
test('some Windows-only installer behavior', {
  skip: process.platform !== 'win32' ? 'Windows only' : false
}, () => { /* ... */ })
```
```python
if platform.system() != "Windows":
    raise unittest.SkipTest("NTFS ACLs are Windows-only")
```
When adding the macOS counterpart to a Windows-only test, mirror this exact skip-string convention with `'macOS only'` / `"macOS-only"` rather than inventing a new gating mechanism — the `test:known-failures`/baseline tooling and the disruptive-probe gate both assume this skip-reason-string shape.

---

*Testing analysis: 2026-09-17*
