# Orgtree (macOS Port)

## What This Is

Orgtree is a desktop workspace for running a persistent team of coding agents (Claude Code, Codex, Antigravity, OpenRouter models), presented as an interactive org chart. This fork ports the existing Windows-only Electron + Python-engine app to run natively on macOS, so the same agent-orchestration workflow works on a Mac.

## Core Value

The app runs and works correctly on macOS: build, launch, spawn agents, and manage their work end-to-end — matching what the Windows build already does.

## Requirements

### Validated

<!-- Inferred from existing Windows codebase — .planning/codebase/ARCHITECTURE.md, STACK.md -->

- ✓ Electron desktop UI renders an interactive org chart of agents — existing
- ✓ Engine (Python, HTTP+bearer-token over loopback) drives agent lifecycle, mail, tickets, evidence — existing
- ✓ Multi-provider agent execution via CLI subprocess spawning (Claude Code, Codex, Antigravity, OpenRouter) — existing
- ✓ Persistent conversation/task history, git worktree-based agent workspaces — existing
- ✓ Windows installer (NSIS), auto-update, boot-time engine autostart via Scheduled Task — existing (Windows only)

### Active

- [ ] **MAC-01**: `package.json` electron-builder config gains a working `mac` target (unsigned, local build)
- [ ] **MAC-02**: App icon assets exist in `.icns` (currently `.ico`-only)
- [ ] **MAC-03**: `tools/provision-runtime.py` provisions a macOS Python runtime (currently hard-exits on non-`win32`)
- [ ] **MAC-04**: All `os.name == "nt"` / `cmd.exe` / `.exe`/`.cmd` executable-resolution branches in `engine/backend/orgtree/` get a macOS/POSIX equivalent
- [ ] **MAC-05**: `ctypes.windll.kernel32` calls (`liveness.py`, `antigravityrun.py`) get a macOS-safe equivalent or platform branch
- [ ] **MAC-06**: Windows `taskkill` process-containment (`gitrunner.py`) gets a POSIX process-group equivalent
- [ ] **MAC-07**: `apps/desktop/main/taskbar.ts` Windows-only Electron APIs (`setAppDetails`, `flashFrame`) get a macOS dock/badge equivalent or safe no-op
- [ ] **MAC-08**: Boot-time engine autostart ported from Windows Scheduled Task + registry to a macOS `launchd` agent
- [ ] **MAC-09**: App builds, launches, and completes a real agent job end-to-end on macOS
- [ ] **MAC-10**: Windows-only test suite (installer/elevation/taskbar probes, `test_service_host.py` skips) gets macOS counterparts so mac-specific logic has real coverage, not silent skips

### Out of Scope

- Signed/notarized `.dmg` distribution — no Apple Developer account yet; local unsigned builds are sufficient for v1
- NSIS installer replacement (e.g., `.pkg` installer) — running via `electron-builder --mac --dir` / unsigned `.app` is enough for v1
- Linux support — not requested, no Linux-specific findings in codebase map

## Context

- This is a fork (`WhoReallyKnowsAnything/orgtree`, `upstream` = `Maurdekye/orgtree`), cloned to `~/Documents/Projects/orgtree`. Original author gave permission to fork; repo is MIT licensed regardless.
- Full existing-codebase analysis lives in `.planning/codebase/` (STACK, ARCHITECTURE, STRUCTURE, CONVENTIONS, TESTING, INTEGRATIONS, CONCERNS — 1064 lines, committed `ea1eeea`). CONCERNS.md enumerates 14 explicit macOS-port blockers with file paths — treat it as the primary source of truth when planning phases.
- No `pywin32` dependency — Windows-specific code uses stdlib `ctypes`/`os.name`, not a native extension, which makes most of it portable to `ctypes` no-ops or `os.name`-branched POSIX code rather than requiring a rewrite.
- Electron main↔Python engine boundary is HTTP+bearer-token over loopback, not stdio — this layer is already platform-agnostic and shouldn't need porting work.

## Constraints

- **Tech stack**: Must stay Electron (TS/JS) + Python engine — matching upstream's stack, no rewrite to a different framework, to keep the fork mergeable/rebasable against upstream.
- **Distribution**: Unsigned local build only for v1 — no Apple Developer account, no notarization pipeline.
- **Compatibility**: Keep Windows build working — this is a fork adding macOS support, not replacing the Windows target, so upstream sync stays possible.

## Key Decisions

| Decision | Rationale | Outcome |
|----------|-----------|---------|
| v1 ships as unsigned local build, not signed .dmg | No Apple Developer account; user just wants it running on their Mac | — Pending |
| Port Windows boot-autostart to launchd rather than dropping it | User wants autostart parity with the Windows build | — Pending |
| Keep full provider parity (Claude Code/Codex/Antigravity/OpenRouter) | No provider-specific blockers found — only the exec-resolution logic around them is Windows-specific | — Pending |

## Evolution

This document evolves at phase transitions and milestone boundaries.

**After each phase transition** (via `/gsd-transition`):
1. Requirements invalidated? → Move to Out of Scope with reason
2. Requirements validated? → Move to Validated with phase reference
3. New requirements emerged? → Add to Active
4. Decisions to log? → Add to Key Decisions
5. "What This Is" still accurate? → Update if drifted

**After each milestone** (via `/gsd-complete-milestone`):
1. Full review of all sections
2. Core Value check — still the right priority?
3. Audit Out of Scope — reasons still valid?
4. Update Context with current state

---
*Last updated: 2026-09-17 after initialization*
