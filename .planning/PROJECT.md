# Orgtree (macOS fork)

## What This Is

Orgtree is a desktop workspace for running a persistent team of coding agents (Claude Code, Codex, Antigravity, OpenRouter models), shown as an interactive org chart. This fork (`WhoReallyKnowsAnything/orgtree`) adds native macOS support to the Windows-only upstream (`Maurdekye/orgtree`) so the same agent-orchestration workflow runs on a Mac.

## Core Value

The app runs and works correctly on macOS (build, launch, spawn agents, manage their work end to end) without breaking the Windows build that upstream ships.

## Requirements

### Validated

- ✓ Electron desktop UI renders an interactive org chart of agents - upstream
- ✓ Python engine (HTTP + bearer token over loopback) drives agent lifecycle, mail, tickets, evidence - upstream
- ✓ Multi-provider agent execution via CLI subprocess spawning (Claude Code, Codex, Antigravity, OpenRouter) - upstream
- ✓ Persistent conversation/task history, git-worktree agent workspaces - upstream
- ✓ Windows installer (NSIS), auto-update, boot autostart via Scheduled Task - upstream
- ✓ Unsigned, ad-hoc-signed macOS `.app` builds with bundled Python runtime and `.icns` icon - GSD phase 1
- ✓ Provider CLI resolution, liveness checks and process-tree termination work on POSIX (process groups, `os.killpg`) - GSD phase 2
- ✓ Engine process lifetime uses a POSIX adapter instead of Windows job objects - GSD phase 2.1
- ✓ Engine autostart via `launchd` LaunchAgent with disabled/crash-loop detection and remediation dialog - GSD phase 3
- ✓ macOS UI parity: native app menu, dock bounce/badge, template tray icon, window recreate, mac update notice - GSD phase 4
- ✓ Real agent turn completes end to end on macOS hardware (VER-01); Windows-only tests have explicit macOS disposition (VER-02) - GSD phase 5

### Active (milestone v2.2.0)

- [ ] Full test suite is green on macOS, including the update-rehearsal tests
- [ ] LaunchAgent autostart confirmed across a real reboot and a real crash loop
- [ ] Fork carries current upstream (`Maurdekye/orgtree` main) with macOS support intact and Windows build still working
- [ ] Signed, notarized `.dmg` distribution
- [ ] macOS auto-update through `electron-updater`
- [ ] GUI toggle for autostart layered on the LaunchAgent

### Out of Scope

- Linux support - not requested; no Linux-specific work in the codebase
- `.pkg` installer - `.dmg` covers distribution; `.pkg` adds nothing a user needs here
- Rewriting away from Electron + Python - would make the fork unmergeable with upstream

## Context

- Stack: Electron main/preload/renderer in TypeScript (`apps/desktop/`, `packages/`), Python engine in `engine/backend/orgtree/`, `engine/mailhub` git submodule, electron-builder packaging, tests under `tests/` (node `--test` `.mjs` and pytest).
- `package.json` version `2.1.7-beta.0`; tags `v2.1.x` come from upstream, latest `v2.1.9`.
- Fork is 122 commits ahead and 369 behind `upstream/main` as of 2026-09-26: upstream sync is a large merge, not a rebase.
- Prior planning history (GSD, phases 1-5) is archived in `.planning-gsd/`, including `codebase/CONCERNS.md` and per-phase research.
- Known rough edges: `tests/update-rehearsal.test.mjs` / `tools/rehearsal-isolation.mjs` fail on POSIX (Windows drive-letter fixtures vs. error message); phase 3 has no VERIFICATION report and its reboot/crash-loop checks were left as human UAT.
- Windows-specific code uses stdlib `ctypes`/`os.name` branches, no `pywin32`, so platform branching stays local.

## Constraints

- **Tech stack**: Electron (TS) + Python engine - matches upstream so the fork stays mergeable.
- **Compatibility**: Windows build must keep working - fork adds macOS, does not replace Windows.
- **Distribution**: Signing and notarization need an Apple Developer account - the distribution work is blocked until one exists.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| Electron + Python engine, no rewrite | Keep fork rebasable/mergeable against upstream | ✓ Good |
| Ship v1 macOS as unsigned local `.app` | No Apple Developer account at the time | ✓ Good (shipped in GSD phase 1) |
| Port boot autostart to a `launchd` LaunchAgent | Autostart parity with Windows Scheduled Task | ✓ Good (GSD phase 3) |
| POSIX process groups + `os.killpg` for process-tree kill | Replaces `taskkill /T` containment | ✓ Good (GSD phase 2) |

---
*Last updated: 2026-09-26 after /cad-adopt*
