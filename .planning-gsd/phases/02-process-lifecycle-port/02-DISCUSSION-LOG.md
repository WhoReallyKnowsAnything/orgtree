# Phase 2: Process Lifecycle Port - Discussion Log

> **Audit trail only.** Do not use as input to planning, research, or execution agents.
> Decisions are captured in CONTEXT.md — this log preserves the alternatives considered.

**Date:** 2026-09-17
**Phase:** 2-Process Lifecycle Port
**Areas discussed:** Claude Code CLI resolution, Data-root guardian scope, Termination signal strategy

---

## Claude Code CLI resolution

| Option | Description | Selected |
|--------|-------------|----------|
| Spawned outside engine/ | Claude Code invocation lives elsewhere (desktop/Electron layer or similar), not the Python engine — Phase 2 doesn't touch it | |
| Build claude_path() now | Add a claude_path() resolver in providers.py mirroring codex_path()/antigravity_path() | |
| You decide | Let research/planning locate where it actually lives first, then handle accordingly | ✓ |

**User's choice:** You decide
**Notes:** Grep across `engine/` found no `claude_path()`/CLI spawn for "claude" (only `codex_path()`, `antigravity_path()`). Claude's resulting call: research repo-wide first; build the resolver only if nothing else already handles it. See CONTEXT.md D-01/D-02.

---

## Data-root guardian scope

| Option | Description | Selected |
|--------|-------------|----------|
| In scope — add macOS branch | Give process_lifetime.py a POSIX equivalent so launch.py doesn't crash on macOS | |
| Out of scope — separate item | Leave process_lifetime.py Windows-only for now; Phase 2 stays strictly PROC-01/02/03 | |
| You decide | Let the planner assess and choose | ✓ |

**User's choice:** You decide
**Notes:** `launch.py:358-361` unconditionally calls `arm_process_lifetime()`, which hard-raises on non-Windows — blocks macOS engine startup entirely, but isn't named in PROC-01/02/03. Claude's resulting call: keep out of Phase 2's committed scope, flag it as a roadmap gap to close before Phase 5's VER-01. See CONTEXT.md D-03/D-04.

---

## Termination signal strategy

| Option | Description | Selected |
|--------|-------------|----------|
| Match gitrunner.py — immediate SIGKILL | os.killpg(pid, SIGKILL), no grace period — same as existing correct pattern | |
| SIGTERM-then-SIGKILL grace period | Give the provider CLI a short window to exit cleanly first | |
| You decide | Let the planner pick | ✓ |

**User's choice:** You decide
**Notes:** Roadmap text itself already says to match gitrunner.py's pattern for PROC-03. Claude's resulting call: immediate SIGKILL, matching gitrunner.py exactly. See CONTEXT.md D-05/D-06.

---

## Claude's Discretion

All three areas above were deferred to Claude ("You decide"). Resulting decisions recorded in CONTEXT.md as D-02, D-04, D-06.

## Deferred Ideas

- Data-root guardian / `process_lifetime.py` POSIX port — recommended as a new roadmap phase or decimal insertion, not part of Phase 2.
- `managed_profiles.py`'s `ctypes.WinDLL` directory-ACL calls — Windows-only but not a process-lifecycle concern; belongs to a different phase.
