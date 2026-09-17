<!-- refreshed: 2026-09-17 -->
# Architecture

**Analysis Date:** 2026-09-17

## System Overview

```text
┌─────────────────────────────────────────────────────────────┐
│                     Electron Renderer (React)                │
│         `apps/desktop/renderer/src/App.tsx` + canvas/*        │
└──────────────────┬─────────────────────────┬──────────────────┘
                    │ ipcRenderer/contextBridge │ fetch/WebSocket (HTTP)
                    ▼                            ▼
┌───────────────────────────────┐   ┌───────────────────────────────┐
│  Preload bridge (contextBridge)│   │  Electron Main Process        │
│  `apps/desktop/preload/index.ts│   │  `apps/desktop/main/index.ts`  │
└───────────────────────────────┘   │  engine.ts, windows.ts,        │
                                     │  taskbar.ts, updater.ts, ...   │
                                     └────────────┬────────────────────┘
                                                  │ spawn(python, launch.py)
                                                  │ HTTP over 127.0.0.1:<port> + bearer token
                                                  ▼
                                     ┌───────────────────────────────┐
                                     │  Python "engine" backend       │
                                     │  `engine/launch.py`             │
                                     │  `engine/backend/orgtree/api.py`│
                                     │  (FastAPI + WebSocket, uvicorn) │
                                     └────────────┬────────────────────┘
                                                  │ file I/O
                                                  ▼
                                     ┌───────────────────────────────┐
                                     │  On-disk org data (JSON/SQLite,│
                                     │  ledger files, per-org dirs)   │
                                     └───────────────────────────────┘
```

## Component Responsibilities

| Component | Responsibility | File |
|-----------|----------------|------|
| Main process bootstrap | App lifecycle, window/tray creation, IPC handler wiring | `apps/desktop/main/index.ts` |
| Engine supervisor | Spawns/attaches/restarts the Python backend, health probing, graceful shutdown | `apps/desktop/main/engine.ts` |
| Window management | BrowserWindow creation, placement persistence, controls state | `apps/desktop/main/windows.ts`, `apps/desktop/main/window-placement.ts` |
| Taskbar integration | Windows taskbar grouping via `setAppDetails` (AppUserModelId) | `apps/desktop/main/taskbar.ts` |
| Taskbar attention/flash | Cross-org "needs attention" flashing on the taskbar/tray | `apps/desktop/main/taskbar-attention.ts` |
| Tray menu | System tray icon, org list popup, engine status menu items | `apps/desktop/main/traylist.ts` (list), wired in `index.ts` (`rebuildTray`) |
| Auto-update | electron-updater wiring, NSIS installer flow, update failure UI | `apps/desktop/main/updater.ts`, `apps/desktop/main/installer-upgrade.ts`, `build/installer.nsh` |
| Preload bridge | Whitelisted `contextBridge` API exposed to renderer (IPC surface) | `apps/desktop/preload/index.ts` |
| Renderer shell | React app root, canvas rendering of org charts | `apps/desktop/renderer/src/App.tsx`, `apps/desktop/renderer/src/Canvas.tsx` |
| Renderer canvas widgets | Org-chart specific UI: docket, pins, accounts, agent menus | `apps/desktop/renderer/src/canvas/*.tsx` |
| Shared cross-package contracts | Types/enums shared between main, preload, renderer | `packages/contracts/*.ts` |
| Python API layer | FastAPI app: org CRUD, ledger ops, event tail, WebSocket "changed" pings | `engine/backend/orgtree/api.py` |
| Python process launcher | Entry point spawned by Electron main; binds to a local port, prints readiness to stdout | `engine/launch.py` |
| Python domain modules | ~107 modules covering agent providers (Claude/Codex/Antigravity/OpenRouter), sandboxing, git integration, notifications, supervisor/lifecycle | `engine/backend/orgtree/*.py` |

## Pattern Overview

**Overall:** Electron desktop app with a **sidecar process architecture** — the Python "engine" is not embedded via a native binding; it is spawned as an independent OS process and talks to Electron's main process exclusively over **local HTTP** (`http://127.0.0.1:<port>` + bearer token), never over stdio pipes for data (stdio is used only to detect readiness/port during boot).

**Key Characteristics:**
- Three-process model: Electron main (Node) ↔ Electron renderer (Chromium, sandboxed) ↔ Python engine (FastAPI/uvicorn), each with a distinct IPC mechanism.
- The renderer talks to main via `contextBridge`-exposed IPC and likely also talks to the Python engine directly over HTTP/WebSocket using the origin+token main hands it — confirm exact wiring in `apps/desktop/renderer/src/api.ts` / `livebus.ts` before assuming.
- Main process is the sole owner of engine process lifecycle (spawn, health-check, restart-on-crash, graceful shutdown for updates/quit) — a supervisor pattern, not fire-and-forget.
- Extremely fine-grained module decomposition in the Python backend (~107 single-purpose files under `engine/backend/orgtree/`) rather than a handful of large service classes — many small, verb/noun-named modules (`halt.py`, `steer.py`, `warmpool.py`).
- Heavy use of docs-as-contracts: `docs/engine-contract.md`, `docs/frozen-turn-lifecycle.md`, and per-release `docs/release-notes-*.md` describe cross-process behavior contracts not otherwise enforced at compile time (TS and Python are separate type systems with no shared schema generation observed beyond the renderer-local `apps/desktop/renderer/src/generated/events.schema.json`).

## Layers

**Electron Main (Node.js):**
- Purpose: OS integration, process supervision, native window/tray/taskbar/updater control.
- Location: `apps/desktop/main/*.ts`
- Contains: lifecycle orchestration, `Engine` class (spawn/attach/restart/stop), Windows-specific shell integration, NSIS installer/updater glue.
- Depends on: `packages/contracts/*` (shared types), Electron/Node APIs, `electron-updater`.
- Used by: renderer (via preload/IPC), OS (installer/updater), Python engine (as its process parent).

**Preload (isolated bridge context):**
- Purpose: Minimal, whitelisted surface exposed to the renderer via `contextBridge`, forwarding `DesktopEvent`s and exposing IPC-safe functions.
- Location: `apps/desktop/preload/index.ts` (50 lines — deliberately thin)
- Depends on: `packages/contracts/index`, `packages/contracts/ui-route`.
- Used by: renderer (`window.<bridge>` global).

**Renderer (React, Chromium sandbox):**
- Purpose: UI — the org-chart canvas, settings, account/agent management screens.
- Location: `apps/desktop/renderer/src/*`
- Contains: React components (`App.tsx`, `Canvas.tsx`), canvas sub-widgets (`canvas/*.tsx`), event decoding (`events/*.ts`), generated event schema (`generated/events.schema.json` + `.ts`).
- Depends on: preload bridge for main-process IPC; direct HTTP/WebSocket to the Python engine's origin+token for live data (verify in `renderer/src/api.ts`).
- Used by: end user directly.

**Python Engine (FastAPI/uvicorn, separate process):**
- Purpose: "The UI's backend" — org CRUD, tree state, ledger operations, event tail, agent-provider orchestration (Claude/Codex/Antigravity/OpenRouter), sandboxing, git integration.
- Location: `engine/backend/orgtree/*.py`, entry point `engine/launch.py`
- Contains: `api.py` (FastAPI routes, 13k+ lines), provider-specific modules (`codexrun.py`, `antigravityrun.py`, `openrouter.py`), process/lifecycle modules (`supervisor.py`, `lifecycle.py`, `startup.py`), sandboxing (`sandbox.py`), notifications (`desktop_notifications.py`).
- Depends on: local filesystem for org data/ledger; a Python runtime bundled at package time (`extraResources` in `package.json` bundles the whole `engine/` dir minus `__pycache__`).
- Used by: Electron main (as supervisor + HTTP client) and renderer (direct HTTP/WebSocket for live data).

**Shared Contracts:**
- Purpose: TypeScript-only types/enums shared across main/preload/renderer so all three agree on event shapes, agent colors, theming, route names.
- Location: `packages/contracts/*.ts` (`index.ts`, `agent-colors.ts`, `contrast-theme.ts`, `notifications.ts`, `ui-route.ts`, `visual-theme.ts`)
- Depends on: nothing (leaf package).
- Used by: `apps/desktop/main/*`, `apps/desktop/preload/*`, `apps/desktop/renderer/*`.
- Note: no equivalent shared-contract mechanism exists between TypeScript and the Python engine — that boundary is documented in prose (`docs/engine-contract.md`) rather than generated from a schema.

## Data Flow

### Primary Request Path (renderer action → persisted org data)

1. User interacts with a canvas widget, e.g. `apps/desktop/renderer/src/canvas/cards.tsx`.
2. Renderer issues an HTTP call (or IPC → main → HTTP relay) to the Python engine's origin (`http://127.0.0.1:<port>`) with the bearer token main obtained from `Engine.attach()`/`Engine.start()` (`apps/desktop/main/engine.ts:154,246`).
3. FastAPI route in `engine/backend/orgtree/api.py` handles the org CRUD/ledger op, persists to disk.
4. Engine's WebSocket pushes a "changed" ping (per `api.py` module docstring, lines 10-11) to connected clients.
5. Renderer refetches/updates state; canvas re-renders.

### Engine Boot / Supervision Flow

1. Electron main calls `Engine.start(options)` (`apps/desktop/main/engine.ts:182-255`), which spawns `python launch.py` with `windowsHide: true, stdio: 'pipe'`.
2. Python process binds a local port and prints readiness info to stdout; `Engine.onData` (`engine.ts:235-249`) parses that to learn the port and set `state: 'ready'`.
3. Main can instead `attach()` to an already-running engine (`engine.ts:131-180`) — used for engine restart-without-relaunch and update flows.
4. On crash, `childExited` fires and the tray/UI reflect `EngineStatus` via `trayEngineState`/`refreshTrayEngineMenu` (`engine.ts:694-715`).
5. On quit or installer upgrade, `stopForQuit`/`stopGracefullyForInstaller` orchestrate graceful HTTP shutdown requests before falling back to `forceKillTree` (SIGKILL on non-Windows; a tree-kill on Windows — verify the win32 branch's full body before assuming parity).

**State Management:**
- Engine process state (`starting/ready/error/restarting`) lives in the `Engine` class instance in main — not persisted, rebuilt on each launch/attach.
- Org/business data lives entirely in the Python engine's on-disk store (ledger files, per-org directories) — Electron holds no independent copy of domain state.
- Renderer UI state (window layout, draft text, notification visibility) uses browser-side stores: `windowlayout.ts`, `draftstore.ts`, `noticestore.ts`.

## Key Abstractions

**`Engine` class (main process):**
- Purpose: Single source of truth for the Python sidecar's lifecycle — spawn, attach, health-verify, restart, graceful/forced shutdown.
- Examples: `apps/desktop/main/engine.ts`
- Pattern: Stateful class wrapping `child_process.spawn`, with async methods returning discriminated-union-like string results (`'attached' | 'spawned' | 'failed'`, `'stopped' | 'forced' | 'unverified'`).

**`DesktopEvent` (cross-process event envelope):**
- Purpose: Typed events broadcast from main to renderer (`broadcast(event: DesktopEvent)` in `index.ts:222`) via the preload bridge's `ipcRenderer.on` handler (`preload/index.ts:44`).
- Examples: `packages/contracts/index.ts`, `apps/desktop/preload/index.ts`
- Pattern: Central event union type shared via the `contracts` package rather than per-channel ad hoc IPC payloads.

**Provider modules (Python engine):**
- Purpose: Each AI provider/harness (Claude, Codex, Antigravity, OpenRouter) gets its own dedicated run/limit/route module set.
- Examples: `engine/backend/orgtree/codexrun.py` + `codex_limits.py` + `codex_route.py` + `codexpin.py`; `engine/backend/orgtree/antigravityrun.py` + `antigravity_limits.py` + `antigravity_session.py`.
- Pattern: Repeated per-provider module quartet (run/limits/route/pin or session) rather than a single polymorphic "Provider" base class — the naming convention lets you find a provider's files by grepping its name prefix.

## Entry Points

**Electron Main:**
- Location: `apps/desktop/main/index.ts` (built to `dist/main/index.cjs`, referenced by `package.json`'s `"main"`)
- Triggers: `electron .` (via `npm start`/`npm run dev`) or the packaged app executable.
- Responsibilities: creates windows, tray, wires all IPC `handle()` channels, starts/attaches the `Engine`, owns app-quit and installer-upgrade shutdown sequencing.

**Renderer:**
- Location: `apps/desktop/renderer/src/main.tsx` (mounts `App.tsx`) via `apps/desktop/renderer/index.html`
- Triggers: loaded into each `BrowserWindow` created by main.
- Responsibilities: renders the org-chart canvas UI, issues data requests to the engine, reflects live `DesktopEvent`s from main.

**Python Engine:**
- Location: `engine/launch.py`
- Triggers: spawned by `Engine.start()` in main, or run manually for dev (`python -m orgtree.api`, or `uvicorn orgtree.api:app --reload --port 7360` with the Vite dev server proxying `/api` — per `api.py`'s header docstring).
- Responsibilities: serves the FastAPI app + built frontend on one port in production; in dev, serves only the API while Vite serves the renderer separately.

## Architectural Constraints

- **Threading:** Python side mixes `asyncio` (FastAPI/uvicorn event loop) with `threading` (imported in `api.py`) — verify per-module before touching concurrency-sensitive code.
- **Global state:** `engine/backend/orgtree/store.py`, `registry.py`, `staffcache.py` and similar modules hold process-wide singletons/caches inside the Python engine process — the engine process is itself a single long-lived global-state container by design (sole source of truth for org data while running).
- **Windows-only shell coupling:** `Engine.forceKillTree` special-cases `process.platform !== 'win32'` to just `process.kill(pid, 'SIGKILL')` (`engine.ts:453`), implying the win32 branch does an OS-specific tree-kill with no direct macOS equivalent — the macOS path is already a plain SIGKILL fallback, so this one is *handled*, but it confirms the codebase defaults to Windows-first with Unix as an afterthought branch.
- **Windows-only taskbar API surface:** `apps/desktop/main/taskbar.ts` calls `window.setAppDetails(...)` — an Electron API documented as a no-op outside Windows. Safe to call on macOS, but provides zero benefit; macOS equivalents (Dock badge/menu via `app.dock`) are not implemented anywhere in `main/`.
- **Windows-only installer/update pipeline:** `package.json`'s `build.win.target` is `nsis` and there is no `build.mac` block at all; `apps/desktop/main/updater.ts` and `installer-upgrade.ts` assume `electron-updater`'s NSIS installer-path/registry-key model (`index.ts:530-605` reads registry-adjacent install-location/log-candidate logic gated on `process.platform === 'win32'`). None of this has a macOS (DMG / Squirrel.Mac) counterpart yet.
- **Windows-only runtime provisioning:** `tools/provision-runtime.py` exits immediately if `sys.platform != "win32"` (line 23) — it provisions a portable embeddable Python (`python.exe`) bundled via `extraResources.from: "engine"`. There is no equivalent runtime-provisioning path for macOS yet; a mac build currently has no bundled-Python-runtime story.

## Anti-Patterns

### Platform checks duplicated ad hoc rather than centralized

**What happens:** `process.platform === 'win32'` / `!== 'win32'` checks are scattered inline across `index.ts` (install-scope, registry reads, updater gating), `engine.ts` (`forceKillTree`), and `harnesses.ts` (harness executable extension logic), each with its own local fallback.
**Why it's wrong:** For a Windows→macOS port, every site is a place the port can silently inherit "whatever the `!== 'win32'` branch happens to do" instead of an intentional macOS behavior — some fallback branches assume Linux/CI, not a real macOS user flow (e.g. install-scope/registry reads simply `resolve()` immediately on non-Windows, so macOS gets no installed-location detection at all, not a macOS-specific implementation).
**Do this instead:** Audit every `process.platform === 'win32'`/`!== 'win32'` branch in `apps/desktop/main/` and decide explicitly whether the non-Windows branch needs a real macOS implementation (updater install-path detection, taskbar/dock parity) or is genuinely fine as a no-op.

### Windows-first bundling with no macOS runtime story

**What happens:** `tools/provision-runtime.py` hard-exits on any non-`win32` platform; `package.json`'s `extraResources` bundles the raw `engine/` Python source tree assuming a `python.exe` sits alongside it via that Windows-only provisioning step.
**Why it's wrong:** There is no equivalent step (e.g. `python-build-standalone` for macOS, py2app, or a vendored interpreter) producing a bundled interpreter for a packaged macOS app — running from source against a system Python only works for local dev, not a shippable `.app`.
**Do this instead:** Add a macOS-equivalent runtime provisioning path before macOS packaging is attempted, and gate a new `package.json` `build.mac` config on it being real.

## Error Handling

**Strategy:** Layered attach/retry with explicit state machines rather than exceptions-only — the `Engine` class encodes recoverable failure as return values (`'attached' | 'spawned' | 'failed'`) and retries with deadlines (`attachWithRetry`, `ATTACH_RETRY_BUDGET_MS`) before surfacing user-facing errors via `showUpdateFailure`/`showUpdateInstallError` in `index.ts`.

**Patterns:**
- Python side: `api.py`'s header comments describe a print-encoding hazard (non-ASCII `print()` calls crashing on Windows' cp1252-redirected stdout) that was only partially guarded — a documented, still-fragile area (`api.py:41-60`). This hazard is Windows-specific (`cp1252`) and likely disappears on macOS (UTF-8 default), but any fix should not assume the encoding is always UTF-8 without checking.
- TS side: engine-lifecycle errors funnel through `recordProcessFailure(stage, detail)` (`index.ts:120-124`) for crash reporting, plus a dedicated `showCrashReports()` UI path.

## Cross-Cutting Concerns

**Logging:** Python engine logs to `backend.log` via redirected stdout (see `api.py` header comment) — subject to the cp1252 encoding hazard above; Electron main logs via `console` (electron-updater itself only logs to `console`, per a comment at `index.ts:106`).
**Validation:** No centralized schema-validation layer observed between renderer and Python engine beyond the renderer-local generated event schema (`apps/desktop/renderer/src/generated/events.schema.json`); FastAPI's type-based request/response validation covers the Python side (`api.py` is `pyright: strict`).
**Authentication:** Bearer token issued by the `Engine` on attach/start and used for all HTTP calls to the local engine port (`engine.ts:96 token()`) — process-local trust, not a real auth system; exists to keep other local processes off the engine's loopback port.

---

*Architecture analysis: 2026-09-17*
