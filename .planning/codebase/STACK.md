# Technology Stack

**Analysis Date:** 2026-09-17

## Languages

**Primary:**
- TypeScript - Electron main/preload/renderer, `apps/desktop/main/`, `apps/desktop/preload/`, `apps/desktop/renderer/src/`
- Python 3.10 - "engine" backend (FastAPI service), `engine/backend/orgtree/`, ~90 modules

**Secondary:**
- JavaScript (`.mjs`) - build/tooling scripts, `tools/*.mjs`
- PowerShell (`.ps1`) - **Windows-only** boot/scheduled-task tooling and installer tests, `tools/*.ps1`, `tests/*.ps1`
- NSIS script (`.nsh`/`.nsi`) - **Windows-only** installer customization, `build/installer.nsh`, `build/installer-dev.nsh`, `build/update-fixture.nsi`

## Runtime

**Environment:**
- Node.js (ESM, `"type": "module"` in `package.json`)
- Electron 44.2.0 (`devDependencies.electron`), entry `dist/main/index.cjs`
- Python 3.10.13 embedded/provisioned runtime for the engine (`tools/provision-runtime.py`, `tools/stage-runtime.mjs`, `tools/runtime-layout.mjs`)

**Package Manager:**
- npm (no yarn/pnpm lockfile evidence checked beyond `package.json`)
- Python deps via `pip`-style `tools/runtime-requirements.in` (pip-compile style, not a full lock)

## Frameworks

**Core:**
- Electron 44.2.0 - desktop shell, `apps/desktop/main/index.ts`
- React 18.3.1 + MUI 9.2.0 + Emotion - renderer UI, `apps/desktop/renderer/src/`
- FastAPI (`>=0.110`) + Uvicorn (`>=0.27`) + Starlette (`>=0.36`) - Python engine HTTP/WS service, `engine/backend/orgtree/api.py`, `engine/service_host.py`
- Pydantic (`>=2.6`) - engine data validation

**Build/Dev:**
- esbuild `^0.28.1` and Vite `8.2.2` - bundling (`tools/build.mjs`)
- TypeScript 7.0.2 (`tsc --noEmit` typecheck only, build uses esbuild/Vite)
- electron-builder 26.15.3 - packaging, **`nsis` target configured only (Windows)** — no `mac`/`dmg` target present in `package.json` `build.win`
- node `--test` runner - JS/TS tests (`tests/*.test.mjs`), no Jest/Vitest dependency found

**Networking/Realtime:**
- `websockets>=12` (Python) for engine↔renderer live updates
- `httpx>=0.27` (Python) for outbound HTTP from the engine

**Other Python deps** (`tools/runtime-requirements.in`, ported from upstream v1):
- `typing_extensions>=4.8`
- `Pillow>=10` - image handling
- `psutil>=7` - process/liveness inspection (cross-platform, but paired with Windows-only `ctypes.windll` calls in places)

## Key Dependencies

**Critical:**
- `electron-updater` `^6.8.0` - auto-update client, configured to publish via GitHub Releases (`package.json` `build.publish`)
- `dompurify` `^3.4.12`, `marked` `^18.0.7` - Markdown/HTML sanitization+rendering in renderer (chat/assistant message content)

**Infrastructure:**
- `@jridgewell/trace-mapping` `^0.3.28` - source map support for build tooling

## Configuration

**Environment:**
- No `.env*` files detected at repo root.
- Engine reads provider CLI paths/executables at runtime (`engine/backend/orgtree/providers.py`) rather than API keys in env vars for most providers; `apikey_accounts.py` exists for API-key-based accounts.

**Build:**
- `package.json` `build` block (electron-builder): appId `com.maurdekye.orgtree`, icon `apps/desktop/assets/orgtree-eye.ico` (**.ico is Windows-only format; no `.icns` for macOS found**), `extraResources` bundles the Python `engine/` directory and `dist/renderer` into the packaged app.
- `tsconfig.json` at root + `apps/desktop/renderer/tsconfig.json` for renderer-specific settings.

## Platform Requirements — macOS Port Flags

**Confirmed Windows-only surfaces requiring porting work:**
- `package.json` `build.win`/`build.nsis` — only NSIS Windows target defined; no `mac`/`dmg`/`zip` target, no `.icns` icon, no code-signing/notarization config for macOS.
- `build/installer.nsh`, `build/installer-dev.nsh`, `build/update-fixture.nsi` — NSIS scripts, Windows installer only.
- PowerShell tooling (`tools/register-boot-engine.ps1`, `tools/unregister-boot-engine.ps1`, `tools/boot-engine-task.ps1`, `tools/boot-task-probe.ps1`, `tools/installer-upgrade.ps1`) — registers/unregisters a **Windows Scheduled Task** (`Register-ScheduledTask`, reads `HKCU:\Environment`) to boot the engine at login. No macOS equivalent (would need `launchd` plist).
- `engine/backend/orgtree/liveness.py:167` — `ctypes.windll.kernel32` (Windows-only ctypes call) for process liveness checks.
- `engine/backend/orgtree/antigravityrun.py:339` — `ctypes.windll.kernel32.GetShortPathNameW`, plus Windows `cmd.exe` argv-escaping logic for hooking commands (`_hook_command`, lines ~368-403).
- `engine/backend/orgtree/gitrunner.py:38` — `subprocess.run(["taskkill", "/PID", ..., "/T", "/F"])` to kill git subprocesses (Windows-only `taskkill`; macOS needs `kill`/`os.kill`).
- `engine/backend/orgtree/providers.py:438,443` — hardcoded `codex.exe`/`codex.cmd` executable names guarded by `os.name == "nt"` branches (also present in `codexrun.py`, `desktop_import_jobs.py`, `frozen_install.py`, and other files — 44 files total across the repo match `os.name == 'nt'` / `sys.platform` per grep, 38 of them under `engine/backend/orgtree/`).
- `providers.py:479` — `argv = (["cmd", "/...` — spawns via Windows `cmd.exe`.
- No `pywin32`/`win32api`/`win32com`/`winreg` package dependency found (checked `runtime-requirements.in` and imports) — Windows-specific behavior is done via stdlib `ctypes.windll` and `os.name` branches, not `pywin32`.
- `apps/desktop/assets/*.ico` — all tray/app icons are `.ico` only; no `.icns`/PNG set for macOS menu bar / dock icon.

**Development:**
- Requires Node.js + npm, Python 3.10, and (for packaging) Windows-specific electron-builder NSIS toolchain.

**Production:**
- Windows desktop install via NSIS-built installer, auto-update via GitHub Releases (`electron-updater`). No macOS distribution path (DMG/notarization) currently configured.

---

*Stack analysis: 2026-09-17*
