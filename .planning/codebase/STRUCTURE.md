# Codebase Structure

**Analysis Date:** 2026-09-17

## Directory Layout

```
orgtree/
├── apps/desktop/            # Electron app (TS/JS)
│   ├── assets/               # .ico icons only (Windows-shaped; no .icns present)
│   ├── main/                 # Electron main process (Node) — 18 files
│   ├── preload/               # contextBridge IPC surface (1 file, deliberately thin)
│   └── renderer/               # React UI
│       ├── src/                 # App shell, canvas/, events/, generated/
│       └── tests/                # co-located renderer tests (.test.tsx) + fixtures
├── packages/contracts/       # Shared TS types/enums (main+preload+renderer)
├── engine/                   # Python backend ("the engine")
│   ├── backend/orgtree/        # ~107 domain modules (flat, no subpackages)
│   ├── docs/charters/           # role/charter markdown for AI agents (business, curator, etc.)
│   ├── launch.py                 # process entry point spawned by Electron main
│   └── mailhub/                  # mail delivery subsystem (empty top dir; check submodule/gitignore)
├── build/                    # NSIS installer scripts (.nsh) — Windows-only packaging assets
├── docs/                     # 65 markdown docs: release notes, engine contract, handoffs
├── tools/                    # build/package/release scripts (.mjs, .py) — see below
├── tests/                    # top-level Node test files (*.test.mjs) incl. disruptive/
├── .github/workflows/         # CI (startup-budget.yml)
├── package.json               # npm workspace root — build config lives here (electron-builder)
└── tsconfig.json
```

## Directory Purposes

**`apps/desktop/main/`:**
- Purpose: Everything that runs in the Electron main (Node) process — OS/window/tray integration, the Python engine supervisor, auto-update.
- Contains: one file per concern, flat (no subfolders): `index.ts` (bootstrap/IPC wiring), `engine.ts` (Python sidecar lifecycle), `windows.ts`/`window-placement.ts` (BrowserWindow), `taskbar.ts`/`taskbar-attention.ts`/`traylist.ts` (Windows shell integration), `updater.ts`/`installer-upgrade.ts`/`build-channel.ts` (auto-update), `harnesses.ts` (AI CLI harness detection), `notifications.ts`, `policy.ts`, `preferences.ts`, `process-failure.ts`, `providerlogin.ts`, `maintenance.ts`, `update-fixture.ts` (test fixture for update flow).
- Key files: `index.ts` (entry point), `engine.ts` (Python IPC boundary).

**`apps/desktop/preload/`:**
- Purpose: The only code allowed to bridge the isolated renderer context and main process.
- Contains: single `index.ts`, exposes a whitelisted API via `contextBridge`.

**`apps/desktop/renderer/src/`:**
- Purpose: React application — org-chart canvas UI and supporting screens.
- Contains: top-level screens/state (`App.tsx`, `Canvas.tsx`, `DiskBrowser.tsx`, `KillSwitch.tsx`, stores like `draftstore.ts`/`noticestore.ts`/`windowlayout.ts`), `canvas/` (59 files — the org-chart widget tree: cards, docket, pins, accounts, agent menus), `events/` (event decode/dedup/project/wire helpers), `generated/` (schema + types generated from the engine's event contract).
- Key files: `App.tsx` (root component), `api.ts` (engine HTTP client — check for direct fetch vs IPC relay), `livebus.ts` (live update subscription).

**`apps/desktop/renderer/tests/`:**
- Purpose: Co-located renderer tests, run via `node apps/desktop/renderer/tests/run.mjs` (custom runner, not a standard jest/vitest config file was found at root).
- Contains: `.test.tsx` files named after the component/feature under test (e.g. `agentcolors.test.tsx`, `accounts-flow.test.tsx`), plus a few `.py`/`.mjs` browser-driven probes (`actlabel_probe.py`, `agentgallery-browser.py`).

**`packages/contracts/`:**
- Purpose: Leaf package of shared TypeScript types consumed by main, preload, and renderer so all three IPC-connected TS contexts agree on shapes.
- Contains: `index.ts` (barrel), `agent-colors.ts`, `contrast-theme.ts`, `notifications.ts`, `ui-route.ts`, `visual-theme.ts`.

**`engine/backend/orgtree/`:**
- Purpose: The Python "engine" — all domain logic: FastAPI API layer, agent-provider orchestration, sandboxing, git integration, notifications, process supervision.
- Contains: ~107 flat `.py` modules, no subpackages — organized entirely by filename convention (see Naming Conventions below), not by directory.
- Key files: `api.py` (FastAPI app, 13k+ lines — the entire HTTP/WebSocket contract), `supervisor.py`/`lifecycle.py`/`startup.py` (process lifecycle), `sandbox.py` (execution sandboxing, has one of the few `darwin`-aware branches in the repo).

**`engine/docs/charters/`:**
- Purpose: Prose "role charter" documents that define behavior contracts for AI agent roles (business, curator, implementer, redteam, coordinator).
- Generated: No. Committed: Yes.

**`build/`:**
- Purpose: NSIS installer script includes (`installer.nsh`, `installer-dev.nsh`) and an update-flow test fixture (`update-fixture.nsi`) — all Windows-installer-specific, referenced from `package.json`'s `build.nsis.include`.
- Generated: No. Committed: Yes. Has no macOS equivalent (no `.plist`/entitlements/notarization scripts anywhere in the repo).

**`docs/`:**
- Purpose: 65 markdown files — per-version release notes (`release-notes-2.1.x*.md`), architecture/contract docs (`engine-contract.md`, `frozen-turn-lifecycle.md`, `mail-delivery-boundaries.md`), and point-in-time handoff/performance notes.
- Naming: `release-notes-<version>[-RCn|-beta.n].md` for release notes; free-form kebab-case for everything else.

**`tools/`:**
- Purpose: Build, packaging, and release automation, mixed `.mjs` (Node/ESM) and `.py` scripts.
- Contains: `build.mjs` (main build), `package-preflight.mjs`, `package-dev.mjs`, `release-windows.mjs` (Windows-specific release — no `release-mac.mjs` exists), `release-verification.mjs`, `provision-runtime.py` (Windows-only Python runtime provisioning, hard-exits on non-win32), `stage-runtime.mjs`, `test-electron.mjs`, `test-baseline.mjs`.

**`tests/`:**
- Purpose: Top-level Node test files run via the built-in `node --test` runner (`npm test` → `node --test tests/*.test.mjs`); `tests/disruptive/` holds a separate, opt-in suite (`npm run test:disruptive`).

## Key File Locations

**Entry Points:**
- `apps/desktop/main/index.ts`: Electron main process bootstrap (built to `dist/main/index.cjs`, referenced by `package.json`'s `"main"`).
- `apps/desktop/renderer/src/main.tsx`: renderer React mount point.
- `engine/launch.py`: Python engine process entry point.

**Configuration:**
- `package.json`: npm scripts, dependencies, and the `electron-builder` `build` block (installer/updater config) all live in one file — no separate `electron-builder.yml`/`.json` exists.
- `tsconfig.json`: root TypeScript config (per-subproject `tsconfig.json` also exists under `apps/desktop/renderer/`).
- `build/installer.nsh`: NSIS include referenced by `package.json`'s `nsis.include`.

**Core Logic:**
- `apps/desktop/main/engine.ts`: the Electron↔Python IPC boundary (spawn/attach/HTTP/token/shutdown).
- `engine/backend/orgtree/api.py`: the entire Python-side HTTP/WebSocket contract.

**Testing:**
- `tests/*.test.mjs`: Node-native top-level tests.
- `apps/desktop/renderer/tests/*.test.tsx`: renderer component tests, run via a custom `run.mjs` harness.
- `tools/test-electron.mjs`, `tools/test-baseline.mjs`: Electron-level and flaky-test-baseline tooling.

## Naming Conventions

**Files:**
- TypeScript/React: lowercase or `camelCase` file names for logic (`engine.ts`, `windowlayout.ts`), `PascalCase.tsx` for React components that export a component as their primary symbol (`App.tsx`, `Canvas.tsx`, `CrashBoundary.tsx`); many smaller widgets stay lowercase even as `.tsx` (`cards.tsx`, `docket.tsx`) — component-name casing is not fully consistent, treat `PascalCase.tsx` vs lowercase `.tsx` as "top-level screen" vs "widget", not a hard rule.
- Python: `snake_case.py` throughout `engine/backend/orgtree/`, with a strong per-provider prefix convention: `codex*.py`, `antigravity*.py`, `openrouter*.py`, `desktop_native*.py`.
- Tests: `<subject>.test.tsx` / `<subject>.test.mjs`, colocated with the code they test (`apps/desktop/renderer/tests/`) or under a top-level `tests/` dir for Node-level tests.

**Directories:**
- Top-level nouns describing the layer (`apps`, `packages`, `engine`, `docs`, `tools`, `tests`, `build`) — a loose monorepo-style layout even though there is a single `package.json` (no workspaces field observed; `packages/contracts` is referenced via relative path, not an npm workspace).
- Python backend intentionally stays flat (`engine/backend/orgtree/*.py`, no subpackages) — new domain modules are added as new files at that same level, not new subdirectories.

## Where to Add New Code

**New Electron main-process feature (e.g. macOS Dock integration):**
- Implementation: new file in `apps/desktop/main/` following the existing one-concern-per-file pattern (e.g. a hypothetical `dock.ts` mirroring `taskbar.ts`'s shape), wired into `index.ts`.
- Shared types (if renderer needs to know about it): `packages/contracts/*.ts`.

**New renderer UI (org-chart widget):**
- Implementation: `apps/desktop/renderer/src/canvas/<name>.tsx` (or `.ts` for non-component helpers), following the flat, lowercase-file convention already used there.
- Tests: `apps/desktop/renderer/tests/<name>.test.tsx`.

**New Python engine capability:**
- Implementation: new flat file in `engine/backend/orgtree/`, named after the capability (follow the per-provider quartet pattern — `run`/`limits`/`route` suffixes — if it's a new AI provider/harness).
- Wiring: register routes/handlers in `engine/backend/orgtree/api.py`.

**New build/packaging step (e.g. macOS runtime provisioning):**
- Script: `tools/<name>.mjs` or `.py`, mirroring the existing Windows-only scripts (`provision-runtime.py`, `release-windows.mjs`) rather than modifying them in place — the Windows scripts are explicitly platform-gated, not shared.
- Wiring: new npm script in `package.json` (mirror `runtime:provision`/`release:windows` naming) and, if packaging output changes, a new `build.mac` block in `package.json`.

**Utilities:**
- Shared TS types/constants: `packages/contracts/`.
- No general-purpose "utils" folder was found in either the TS or Python side — utility functions live inline in the module that needs them, consistent with the flat, single-purpose-file convention.

## Special Directories

**`engine/mailhub/`:**
- Purpose: Listed as a 0-file directory in the tree scan — likely populated at runtime, gitignored, or a submodule mount point rather than committed source. Verify before assuming it's empty by design.
- Generated: Unclear — check `.gitignore`/`.gitmodules` before treating as safe to delete.
- Committed: Unclear (see above).

**`release/`:**
- Purpose: `electron-builder`'s configured output directory (`directories.output` in `package.json`), where `.exe`/NSIS installer artifacts land.
- Generated: Yes. Committed: No (build output).

**`dist/`:**
- Purpose: Build output referenced by `package.json`'s `"main"` (`dist/main/index.cjs`) and `files`/`extraResources` entries (`dist/renderer`, `dist/build-info.json`).
- Generated: Yes (via `npm run build` → `tools/build.mjs`). Committed: No.

---

*Structure analysis: 2026-09-17*
