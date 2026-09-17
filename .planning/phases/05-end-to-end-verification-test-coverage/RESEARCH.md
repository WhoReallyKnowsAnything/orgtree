# Phase 5: End-to-End Verification & Test Coverage - Research

**Researched:** 2026-09-17
**Domain:** macOS port acceptance/E2E proof + closing Windows-only test skips (Node `node:test` + Python `unittest`)
**Confidence:** MEDIUM

## Summary

Phase 5 is the last phase of a 5-phase macOS port roadmap, and **none of Phases 1-4 have been started** (`.planning/STATE.md`: `completed_phases: 0`, current focus = Phase 1; `package.json`'s `build` config has only a `win` target, no `mac` target at all). Every claim in this document about what Phase 1-4 will produce is therefore taken from `.planning/ROADMAP.md`'s stated success criteria, not from inspecting real Phase 1-4 output — those artifacts do not exist yet in this repo. This is flagged inline everywhere it matters.

What *does* exist and was verified by reading the actual source: (1) a precise, git-committed inventory of every Windows-only test skip already exists in `.planning/codebase/TESTING.md`, (2) an existing `tests/acceptance/` harness (`run.mjs`, `application.cjs`, `isolation.mjs`) that launches the real instrumented Electron app end-to-end but **explicitly excludes a real provider/agent turn** ("No real provider turn in this suite" is a documented limit in `run.mjs`'s own summary object), and (3) at least one Windows-only process-tree-kill bug pattern (`codexrun.py::CodexProcess.close()`) that is directly reachable from a real end-to-end agent run and will need fixing before VER-01's "no manual workarounds" bar can be met on macOS, regardless of which phase fixes it.

The `mcp__macos-harness` MCP tool named in the task brief was **not available as a callable tool in the research session** — no `mcp__macos-harness__*` tool appeared, and no reference to it exists anywhere in the repo (searched 1202 files). Recorded as an open question, not a fact — the planner should verify on the actual execution host.

**Primary recommendation:** Phase 5's VER-01 test cannot reuse `tests/acceptance/run.mjs` as-is — that harness deliberately stubs out the provider process. Plan a *new* acceptance runner (same pattern: instrumented Electron + isolated data root + structured JSON report) that spawns a real (or minimally-faked, but real-process) provider CLI and asserts the resulting mail/ledger entry lands in the org tree, then wire the existing Windows-only skips over to real macOS-branch tests using the codebase's own documented skip-string convention (`'macOS only'` mirroring `'Windows only'`).

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Package/launch a real `.app` and assert it starts | Electron main process | — | `tests/acceptance/run.mjs` already drives this; Phase 5 extends it, doesn't relocate it |
| Spawn a real agent (Codex/Claude/Antigravity CLI) and observe its output | Python engine (`engine/backend/orgtree/*run.py`) | Electron main (spawns `python launch.py`) | Agent spawn/lifecycle is entirely engine-side |
| Result "lands back in Orgtree" (mail/ledger write visible in UI) | Python engine (ledger/store write) | Renderer (reads via HTTP/WS) | Ledger write happens in the Python engine |
| Windows-only test skip -> macOS counterpart (NTFS ACL / SDDL tests) | Python `unittest`, `engine/service_host.py` | — | POSIX-permission-model tests; no UI tier involved |
| Windows-only test skip -> macOS counterpart (installer elevation) | Build/Packaging (NSIS-specific) | — | No macOS tier owns this — see Pitfall 1: may be no macOS equivalent to build at all |
| Process-tree kill correctness under a real agent run | `codexrun.py`, `antigravityrun.py`, `gitrunner.py` | — | Backend process lifecycle, not UI |

## Standard Stack

No new external packages required. Reuses frameworks already present and verified:

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| `node:test` (Node builtin) | Node 24.x | TS/JS unit + acceptance test runner | Already the project's only test runner for TS; no jest/vitest/mocha |
| `unittest` (Python stdlib) | Python 3.10.13 | Python backend test runner | Already the project's only Python test framework |
| `electron` | 44.2.0 `[VERIFIED: package.json:34]` | Driver for acceptance-harness Electron process | Already used by `tests/acceptance/run.mjs` |
| `electron-builder` | 26.15.3 `[VERIFIED: package.json:35]` | Packages the `.app` this phase must prove launches | Phase 1 is expected to add a `mac` target to this tool, not a new one |
| `tools/test-baseline.mjs` (in-repo) | n/a | Tracks a "known failures" baseline per platform instead of a coverage percentage | Already the project's mechanism for macOS-port test gaps — reuse rather than inventing a coverage gate |

**Alternatives considered:** Playwright/Spectron/WebdriverIO for Electron E2E — none exist in the repo today; introducing one for Phase 5 alone would add a net-new framework dependency against this project's existing pattern of reusing `node:test` everywhere. Not recommended.

**Installation:** No new packages required — this phase reuses existing devDependencies.

## Package Legitimacy Audit

Not applicable — this phase installs no new external packages.

## Architecture Patterns

### System flow (VER-01 acceptance runner)
```text
 [Phase 5 test entry points]
        |
        |--> npm test (node --test tests/*.test.mjs)             -- existing TS suite, extend with macOS branches
        |--> npm run test:renderer                                -- existing renderer suite
        |--> python -m unittest (tests/test_*.py)                 -- existing Python suite, extend with macOS branches
        |--> NEW: real-agent acceptance runner (tests/acceptance/) -- the VER-01 proof
        v
 [tests/acceptance/run.mjs pattern: isolated data root + real instrumented Electron]
        |
        |  spawns real Electron main process (apps/desktop/main/index.ts)
        v
 [Electron main] --spawn(python, engine/launch.py)--> [Python engine, FastAPI/uvicorn]
        |                                                     |
        | HTTP/WS to 127.0.0.1:<port> + bearer token          | spawns provider CLI
        |                                                     v
        |                                          [engine/backend/orgtree/codexrun.py |
        |                                           antigravityrun.py | claude-pipe lane]
        |                                                     |
        |                                          real (or realistic-fixture) agent turn runs
        |                                                     |
        |                                          result written to ledger/store
        v                                                     |
 [Structured JSON report: report.json, phase results] <-------+
        |
        v
 [assert: report.status === 'PASS' AND ledger/mail entry matches expected agent output]
```

Entry point is the acceptance-runner CLI invocation; decision points are (a) whether the app launches without an AMFI/Gatekeeper failure (Phase 1 concern, tested here), (b) whether the spawned provider process and its full descendant tree are correctly torn down (Phase 2 concern, exercised here under real load), and (c) whether the ledger write is observable before test teardown.

### Recommended structure
```
tests/
├── acceptance/            # existing harness — extend, do not fork
│   ├── run.mjs             # existing: launches app, collects report.json; no provider turn today
│   ├── application.cjs     # existing: drives the Electron process from inside
│   └── run_<new>.mjs        # NEW: real-agent end-to-end runner (VER-01)
├── test_*.py               # existing Windows-only-skipped tests to receive macOS counterparts (VER-02)
└── disruptive/             # existing Windows-only destructive probes — mostly N/A on macOS (Pitfall 1)
```

### Pattern: Platform-Conditional Test (established convention)
Gate a test to a single platform using a `skip` reason string, not a hard `if` around the whole file.
```javascript
// Source: tests/installer-elevation.test.mjs:229
test(
  'the helper refuses while anything still runs from the installation folder',
  { skip: process.platform !== 'win32' ? 'Windows only' : false },
  () => { /* ... */ },
)
```
```python
# Source: tests/test_service_host.py:78-80
def test_descriptor_acl_restriction_leaves_owner_system_admins_only(self):
    if os.name != "nt":
        raise unittest.SkipTest("NTFS ACLs are Windows-only")
```
`.planning/codebase/TESTING.md:183` directs: mirror this exact skip-string convention with `'macOS only'` rather than inventing a new gating mechanism — `test:known-failures`/baseline tooling and the disruptive-probe gate assume this skip-reason-string shape.

### Anti-Patterns to Avoid
- Reusing `tests/acceptance/run.mjs` unmodified as the VER-01 proof — its own summary lists `'No real provider turn in this suite'` `[VERIFIED: tests/acceptance/run.mjs:116]`.
- Trying to make `tests/test_service_host.py`'s Windows-only tests "pass" via Wine/emulation instead of writing real POSIX-permission companion tests — explicitly warned against in `.planning/codebase/TESTING.md:153`.
- Porting `tests/installer-elevation.test.mjs` line-for-line to macOS — it asserts against `powershell.exe`, WMI/CIM process enumeration, NSIS installer macros, none of which exist on macOS (Pitfall 1).

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Isolated, reproducible acceptance test environment | New sandbox/mocking layer | `tests/acceptance/isolation.mjs` (`isolatedRoot`, `acceptanceEnvironment`, `assertIsolatedEnvironment`, `preflightHelpers`) | Already exists, already used by `run.mjs` |
| Cross-platform "is this process alive" check | New PID-probe helper | `engine/backend/orgtree/liveness.py::_observe_pid` — has a real POSIX branch (`os.kill(pid, 0)`) `[VERIFIED: engine/backend/orgtree/liveness.py:159-198]` | PROC-02 says *verify* this existing branch, not write a new one |
| Tracking macOS-port test gaps without a hard coverage gate | New coverage-percentage CI gate | `tools/test-baseline.mjs` (`test:known-failures`/`test:compare`/`test:baseline:record`) | Project deliberately chose a known-failures baseline over coverage percentage |

**Key insight:** Almost everything Phase 5 needs (isolation helpers, a POSIX liveness branch, a known-failures baseline mechanism, a documented skip-string convention) already exists. The phase's real work is *filling gaps* (a real-agent acceptance runner, macOS-branch companion tests), not building test infrastructure from scratch.

## Runtime State Inventory — Windows-only test skips (read directly from source, not grepped/guessed)

| File | Line(s) | What it tests | macOS-equivalent needed |
|------|---------|----------------|----------------------------------------------------|
| `tests/test_service_host.py` | 79-80 | `restrict_descriptor_acl` sets owner-only NTFS DACL (icacls) | POSIX: `os.chmod(descriptor, 0o600)` — note `engine/service_host.py:123-144` currently hard-returns `False` on non-`nt`, a real no-op gap, not just an untested branch |
| `tests/test_service_host.py` | 98-99 | Published descriptor carries restricted ACL (SIDs) | POSIX: descriptor file mode `0600`, owned by current uid — no SID concept on macOS |
| `tests/test_service_host.py` | 116-117 | Published descriptor owned by operator (NTFS ownership) | POSIX: `os.stat(descriptor).st_uid == os.getuid()` |
| `tests/test_service_host.py` | 134-136 | `create_protected_exclusive`: restrictive DACL at birth, share mode 0 | POSIX: `os.open(path, os.O_CREAT \| os.O_EXCL, 0o600)` |
| `tests/test_service_host.py` | 154-155 | Verification refuses descriptor owned by someone else | POSIX: same via `st_uid` check |
| `tests/test_service_host.py` | 231-232, 250-251 | Protected birth denies second handle; no residue | POSIX: `O_EXCL` open failing (`FileExistsError`) + residue check |
| `tests/test_service_host.py` | 353-354, 381-382, 416-417 | Boot-host "guardian" tests gated on `arm_process_lifetime` requiring Windows | Blocked on Phase 2/3: `engine/process_lifetime.py:21-29` hard-raises `RuntimeError(...)` on non-`nt` today — nothing to test until a POSIX guardian ships |
| `tests/test_claude_pipe_lifecycle.py` | 215 | Idle watchdog kills launcher+child via Windows `.cmd` wrapper | macOS: same kill path via POSIX shell wrapper/bare exec |
| `tests/test_claude_pipe_lifecycle.py` | 242 | Expiry releases pipe reader with inherited-pipe cleanup (Windows handle inheritance) | macOS: own reproduction — POSIX fd-inheritance failure mode (`close_fds`/`start_new_session`) is not a direct 1:1 port |
| `tests/test_managed_profile_permissions.py` | 14 | Desktop token persistence (Windows token/ACL regression, needs admin token) | No direct analog — needs its own test if/when a macOS-specific security model is built |
| `tests/test_process_lifetime.py` | 41 | Windows Job-object process-tree ownership; POSIX path "not exercised here" per file's own comment | POSIX test needed once Phase 2/3 implement `start_new_session`+process-group guardian — currently no test exists at all to skip |
| `tests/test_startup_progress.py` | 28 | Boot-host progress under guardian — comment says "INERT" | Blocked on Phase 2/3 guardian |
| `tests/test_startup_readiness.py` | 23 | Startup readiness budget under real guardian — comment says "INERT" | Blocked on Phase 2/3 |
| `tests/installer-elevation.test.mjs` | 229, 344, 384, 498, 593, 773, 812, 842, 886 (9 gates) | NSIS UAC elevation ordering, WMI/CIM process enumeration, PowerShell 5.1 presence | Likely **no macOS equivalent to write** — v1 ships unsigned local `.app` with no separate installer (PKG-01/PKG-03); see Pitfall 1 |
| `tests/disruptive/nsis-destination.test.mjs`, `installer-log.test.mjs`, `console-lifetime.test.mjs`, `lifetime-controls.test.mjs` | whole files | NSIS `.exe` installer, Windows console/window semantics | No macOS analog yet (no `.pkg`/`.dmg`/notarization scaffolding); out of v1 scope |
| `tests/disruptive/boot-installer.test.ps1` | whole file | Literal PowerShell script | Cannot execute on macOS; needs `.sh` equivalent or removal |

**Excluded from this table (not test skips):** `tools/release-verification.mjs:247`, `tools/release-windows.mjs:125`, `tools/provision-runtime.py:23` — release/build-tooling platform branches, not test skips.

## Common Pitfalls

### Pitfall 1: Assuming every Windows-only test has a 1:1 macOS counterpart
A plan that tries to "port" `tests/installer-elevation.test.mjs` or NSIS-specific `tests/disruptive/*` line-for-line will fail — no NSIS, no PowerShell 5.1, no WMI/CIM on macOS, and v1 doesn't even have a separate installer program per PKG-01/PKG-03. Classify each skip (table above) into "needs a real POSIX-behavior counterpart" vs "tests a Windows-only mechanism with no macOS analog" before planning tasks; document the latter as "N/A on macOS, closed intentionally" rather than force-porting.

### Pitfall 2: Verifying VER-01 with the existing acceptance harness, believing it's sufficient
`tests/acceptance/run.mjs` looks fully e2e (launches real Electron app, checks two phases, verifies runtime-manifest integrity) but its own `limits` array says `'No real provider turn in this suite'` `[VERIFIED: tests/acceptance/run.mjs:116]`. Plan a distinct runner (or an addition explicitly opting a real/near-real provider path back in) that spawns a provider CLI and asserts the resulting mail/ledger entry.

### Pitfall 3: Process-tree kill bugs surfacing during the real end-to-end run, not in unit tests
`codexrun.py::CodexProcess.close()` spawns via `subprocess.Popen(...)` with no `start_new_session` `[VERIFIED: engine/backend/orgtree/codexrun.py:506-511]`, then `self.proc.kill()` on close `[VERIFIED: codexrun.py:954-957]` — on POSIX this only signals the direct child, not the process group. Compare `gitrunner.py:72`, which does this correctly: `start_new_session=os.name != "nt"`. A second, structurally identical gap exists in `engine/backend/orgtree/antigravityrun.py::kill_tree` (567-584): Windows-only `taskkill /T` branch, no POSIX process-group kill. Per this project's own convention ("patch every occurrence"), if Phase 2 or Phase 5 fixes `codexrun.py`, the same fix must land in `antigravityrun.py`. Technically PROC-03 (Phase 2) scope, but Phase 5's real-agent e2e test is exactly where an unfixed instance surfaces as flaky/failing — flag as a pre-flight dependency check for the planner, not something to silently work around.

## Code Examples

```javascript
// Source: tests/acceptance/run.mjs:96-116
for (const phase of ['initial', 'restart']) {
  const result = spawnSync(electron, acceptanceLaunchArgs(path.join(here, 'application.cjs')), {
    cwd: target, env: { ...env, ORGTREE_ACCEPTANCE_PHASE: phase }, windowsHide: true,
    encoding: 'utf8', timeout: 150000, maxBuffer: 1024 * 1024,
  })
  const reportFile = path.join(root, phase + '.json')
  const report = fs.existsSync(reportFile) ? JSON.parse(fs.readFileSync(reportFile, 'utf8')) :
    { status: 'FAIL', reason: 'Application produced no acceptance report', processStatus: result.status }
}
```
A VER-01 runner follows this same shape (isolated root, structured `report.json`, explicit `limits` array) but adds a phase that triggers a real agent turn and asserts on the resulting ledger/mail write.

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | Phase 1 ships a `.app` with no separate installer program (drag-launch/right-click-Open only) — `installer-elevation.test.mjs` and NSIS-specific `disruptive/*` have no macOS counterpart to write | Runtime State Inventory, Pitfall 1 | If Phase 1 instead ships a `.pkg`/`.dmg` with its own elevation semantics, VER-02 needs real macOS installer-elevation tests — based on ROADMAP/REQUIREMENTS wording, not real Phase 1 code (doesn't exist yet) |
| A2 | Phase 2/3 implement the POSIX "guardian" using `start_new_session=True` + `os.killpg` (standard POSIX pattern) | Runtime State Inventory (`test_process_lifetime.py`, startup rows) | If a different POSIX mechanism is chosen, recommended macOS test counterparts need different assertions — `process_lifetime.py:28-29` only hard-raises today, no real implementation exists yet |
| A3 | `mcp__macos-harness` (`build_app`/`launch_app`/`ax_tree`) exists as an MCP server on the execution host, despite absence from this research session's tool list | Environment Availability | If it genuinely doesn't exist, plan needs the fallback: `spawnSync`+AppleScript/`osascript`, matching `run.mjs`'s existing pattern |
| A4 | The `codexrun.py`/`antigravityrun.py` process-tree-kill gap (Pitfall 3) is still present when Phase 5 executes — i.e. Phase 2 (PROC-03) fixes only the literally-named `codexrun.py` instance, not `antigravityrun.py` too | Pitfall 3 | If Phase 2 already patches both, Phase 5 doesn't need its own pre-flight check; if missed, an Antigravity-provider e2e run could fail non-deterministically |

## Open Questions (RESOLVED)

1. **Is `mcp__macos-harness` actually available on the execution host?** (RESOLVED) Not in this session's tool list; zero repo references. Resolution: 05-01 adopts the recommended fallback — the `spawnSync(electron, ...)` + structured-report / DOM-driven pattern already proven in `run.mjs` — instead of depending on the unconfirmed tool.
2. **What does "a real agent job" mean concretely for VER-01 — a live provider (real network calls) or a scripted fixture CLI mimicking one?** (RESOLVED) `tests/desktop.test.mjs` uses fixture executables for resolution tests; `test_claude_pipe_lifecycle.py` uses a fully scripted fake CLI (`_CHILD` inline script) to reproduce a real incident without a live provider. Resolution: 05-01 implements the recommended split — real-process/fixture-content behind an explicit authorization flag for the automated test, with a fully-live-provider run left as a separate manual `checkpoint:human-verify` step on real Apple Silicon hardware, consistent with how PKG-03 (Gatekeeper approval) is already handled.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Node.js | All TS test suites | Unconfirmed (shell probing capacity-blocked this session) | `@types/node: ^24.0.0` (package.json) | — |
| Python 3.10 | Backend `unittest` suite | Unconfirmed | 3.10.13 per `.planning/codebase/STACK.md` (doc claim) | — |
| `electron`/`electron-builder` | Packaging + acceptance harness | Unconfirmed | 44.2.0 / 26.15.3 `[VERIFIED: package.json:34-35]` | — |
| `mcp__macos-harness` MCP tool | Possible VER-01 driver | Not present in research session's tool list | — | `spawnSync(electron, ...)` pattern in `run.mjs` |

**Missing dependencies with no fallback:** None. **With fallback:** `mcp__macos-harness` → existing `spawnSync` pattern.

*Note: live shell version probes (`node --version`, `python3 --version`) could not be executed — sandbox mutating-tool capacity was saturated throughout research. Planner/executor should re-probe live versions before relying on them.*

## Validation Architecture

| Property | Value |
|----------|-------|
| Framework | Node `node:test` (TS) + Python `unittest` (backend) — no config file for either |
| Quick run | `npm test` (TS) / `python -m unittest tests.test_<name>` (Python) |
| Full suite | `npm test && npm run test:renderer && npm run test:disruptive` + full `unittest` discovery |

### Phase Requirements -> Test Map

| Req ID | Behavior | Test Type | Command | File Exists? |
|--------|----------|-----------|---------|-------------|
| VER-01 | Package, launch, spawn agent, agent works, result lands back in Orgtree, no manual workarounds | acceptance/e2e | `node tests/acceptance/run_e2e_agent.mjs` (name TBD) | No — nearest is `run.mjs`, which excludes a real provider turn |
| VER-02 (ACL/ownership) | macOS POSIX-permission counterparts to `test_service_host.py`'s 6 Windows-only tests | unit | `python -m unittest tests.test_service_host` | File exists; macOS branches don't |
| VER-02 (process-tree/idle-watchdog) | macOS counterparts to `test_claude_pipe_lifecycle.py`'s 2 Windows-only tests | unit | `python -m unittest tests.test_claude_pipe_lifecycle` | File exists; macOS branches don't — blocked on Pitfall 3 fix |
| VER-02 (guardian/lifetime) | macOS counterparts to `test_process_lifetime.py`, `test_startup_progress.py`, `test_startup_readiness.py` | unit/integration | `python -m unittest tests.test_process_lifetime` etc. | Blocked entirely on Phase 2/3 POSIX guardian (A2) |
| VER-02 (installer/elevation) | Scope decision: document as N/A on macOS | manual-only | n/a | Recommend documenting rather than building — Pitfall 1 |

### Wave 0 Gaps
- [ ] `tests/acceptance/run_<new>.mjs` — real-agent e2e runner satisfying VER-01; doesn't exist yet
- [ ] macOS-gated test methods in `tests/test_service_host.py` (6 tests) — don't exist yet
- [ ] macOS-gated test methods in `tests/test_claude_pipe_lifecycle.py` (2 tests) — blocked on Phase 2 process-tree-kill fix (Pitfall 3) landing first
- [ ] Documented, explicit scope decision (not a test file) for `installer-elevation.test.mjs` and NSIS-specific `disruptive/*` — capture in PLAN.md rather than leaving implicit

## Security Domain

| ASVS Category | Applies | Standard Control |
|---------------|---------|-------------------|
| V4 Access Control | yes | macOS counterparts to `test_service_host.py`'s ACL/ownership tests are exactly an access-control verification (owner-only file permissions replacing NTFS DACLs) |

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|----------------------|
| Descriptor file (carries a bearer token per `service_host.py`'s docstring) readable by another local user | Information Disclosure | POSIX: `os.open(path, os.O_CREAT \| os.O_EXCL, 0o600)` at creation, verified via `os.stat().st_mode` in macOS test — mirrors existing Windows intent `[VERIFIED: engine/service_host.py:123-193]` |
| Orphaned provider-CLI descendant retaining a write lock after parent killed | Denial of Service (next turn fails) | `start_new_session=True` at spawn + `os.killpg(os.getpgid(pid), signal.SIGKILL)` at teardown — POSIX equivalent of existing `taskkill /T` (Pitfall 3) |

## Sources

**Primary (HIGH confidence):** Direct source reads this session (all `[VERIFIED: path:lines]` citations above): `.planning/ROADMAP.md`, `.planning/REQUIREMENTS.md`, `.planning/STATE.md`, `.planning/config.json`, `package.json`, `.planning/codebase/TESTING.md`, `.planning/codebase/ARCHITECTURE.md`, `.planning/codebase/STACK.md`, `tests/test_service_host.py`, `tests/test_claude_pipe_lifecycle.py`, `tests/test_managed_profile_permissions.py`, `tests/test_process_lifetime.py`, `tests/test_startup_progress.py`, `tests/test_startup_readiness.py`, `tests/installer-elevation.test.mjs`, `tests/acceptance/run.mjs`, `engine/service_host.py`, `engine/backend/orgtree/liveness.py`, `engine/backend/orgtree/codexrun.py`, `engine/backend/orgtree/antigravityrun.py`, `engine/backend/orgtree/gitrunner.py`, `engine/process_lifetime.py`.

**Secondary/Tertiary:** None — no web search performed; research is entirely repo-internal per task brief.

## Metadata

**Confidence breakdown:**
- Windows-only skip inventory: HIGH — every entry read directly from source, cross-checked against `.planning/codebase/TESTING.md`.
- Phase 1-4 artifact assumptions: LOW — none of Phases 1-4 exist yet; flagged as assumptions (A1, A2).
- `mcp__macos-harness` availability: unknown — could not confirm present/absent from within research session.
- Environment (Node/Python live versions): LOW — shell probing was capacity-blocked; versions are doc/config-file claims, not live probes.

**Research date:** 2026-09-17
**Valid until:** Re-check (not just re-date) once Phases 1-4 have real artifacts — large parts (A1, A2, Wave 0 gaps) are currently assumption-based about that work.
