# Phase 2: Process Lifecycle Port - Context

**Gathered:** 2026-09-17
**Status:** Ready for planning

<domain>
## Phase Boundary

The Python engine correctly finds, runs, monitors, and terminates provider CLI subprocesses on macOS: executable resolution (`providers.py`, `supervisor.py`), liveness checks (`liveness.py`), and process-tree termination (`gitrunner.py`, `codexrun.py`, `antigravityrun.py`, `mailhub_runtime.py`, `supervisor.py`). Covers PROC-01, PROC-02, PROC-03 only — no new capabilities.

</domain>

<decisions>
## Implementation Decisions

### Claude Code CLI resolution (PROC-01)
- **D-01:** No `claude_path()`-style resolver or CLI subprocess spawn for "claude" exists anywhere in `engine/` today — only `codex_path()` (providers.py:447) and `antigravity_path()` (providers.py:883) do this, both with an env-override > private-pin > `shutil.which()` PATH fallback chain. `clipin.py` is only a version-pin constant, not a resolver. User deferred this to Claude's discretion (see below) rather than picking a path themselves — decision made below is Claude's call, not the user's.
- **D-02 (Claude's call):** Research first (gsd-phase-researcher, repo-wide search, not just `engine/`) to confirm whether Claude Code is actually spawned as a subprocess somewhere outside `engine/` (e.g. desktop/Electron layer) or handled via a non-CLI mechanism entirely. If no such spawn exists anywhere, build a `claude_path()` resolver in `providers.py` mirroring the existing `codex_path()`/`antigravity_path()` fallback-chain pattern, as part of this phase's PROC-01 work. If it exists elsewhere, PROC-01 for Claude Code is satisfied by verifying that existing code on macOS, not building new code — no engine-side change needed for it. — **Reversibility:** reversible (isolated to one new function if built; no other code depends on it yet)

### Data-root guardian scope
- **D-03:** `launch.py:358-361` unconditionally calls `process_lifetime.arm_process_lifetime()`, which hard-raises `RuntimeError` on any non-Windows OS (`os.name != "nt"` check, `ctypes`/`WinDLL`/`advapi32`-based data-root exclusive-lock mechanism). This is not named in PROC-01/02/03 but blocks engine startup entirely on macOS — meaning Phase 2's other fixes cannot be exercised end-to-end against a running engine on real hardware, and Phase 5's VER-01 (real end-to-end agent job on macOS) cannot pass until this is closed by *some* phase. User deferred this to Claude's discretion.
- **D-04 (Claude's call):** Keep this **out of Phase 2's committed scope.** Phase 2 stays strictly PROC-01/02/03 (CLI resolution, liveness, termination) per the roadmap's own goal statement, which does not mention data-root locking — pulling in a WinDLL-replacement locking mechanism is a materially different, more architecturally involved piece of work (exclusive-lock ownership across engine crashes) than the three narrow, already-well-specified process-lifecycle fixes this phase targets. **Flag this loudly instead of silently absorbing or silently ignoring it:** recommend the roadmap gain a new phase (or a decimal insertion, e.g. `1.1` or `2.1`) titled something like "Data-Root Ownership Port" before Phase 5's VER-01 is attempted, since without it the packaged `.app` cannot launch the engine on macOS at all regardless of Phase 2's fixes. — **Reversibility:** reversible (a scope note, not a code change; the gap itself is orthogonal and doesn't block *planning* Phase 2, only *end-to-end verification* of it)

### Termination signal strategy (PROC-03)
- **D-05:** `gitrunner.py:36-48`'s existing POSIX branch does an immediate `os.killpg(proc.pid, signal.SIGKILL)` with no grace period — this is the pattern the roadmap explicitly says to match ("matching the pattern already correct in `gitrunner.py`"). User deferred to Claude's discretion, but the roadmap text itself already answers this.
- **D-06 (Claude's call):** `codexrun.py::CodexProcess.close()` and `antigravityrun.py`'s termination path copy `gitrunner.py`'s exact immediate-`SIGKILL` behavior (`os.killpg(pid, signal.SIGKILL)`, no `SIGTERM`-then-wait grace period) — consistent with the existing correct pattern, with Windows `taskkill /T /F`'s equivalent forcefulness, and with what PROC-03 asks for. Both call sites also need `start_new_session=True` added to their `Popen(...)` calls first (`codexrun.py` is missing it per PROC-03; verify `antigravityrun.py`'s `Popen` at antigravityrun.py:756 has it too). — **Reversibility:** reversible (signal choice is a one-line change, not load-bearing elsewhere)

### Supervisor scope expansion (PROC-03, post-research)
- **D-07:** Research (02-RESEARCH.md) found that `supervisor.py` spawns the actual Claude Code CLI at 4 sites (lines ~183, 1234, 2401, 2430) — none pass `start_new_session=True`, and termination goes through `_wd_kill_tree`/`_reap_orphans`/bare `proc.kill()`/`proc.terminate()`, never `os.killpg()`. This is the production "stop agent" entry point (`warmpool.py:2376` `halt_kill()` → `sup._wd_kill_tree()`) for the highest-traffic provider — not named in the original phase boundary, discovered only during research. User decision: **expand PROC-03 to cover `supervisor.py`'s 4 spawn/kill sites**, matching D-06's immediate-SIGKILL-via-`os.killpg()` pattern after adding `start_new_session=True`. This is now in scope for Phase 2, alongside `codexrun.py`/`antigravityrun.py`/`mailhub_runtime.py`. — **Reversibility:** reversible (same one-line-per-site pattern as D-06, no new dependencies)

### Claude's Discretion
All three areas above were explicitly deferred to Claude ("You decide") — see D-02, D-04, D-06 for the resulting calls and rationale. The planner and researcher should treat these as decided, not re-open them with the user.

</decisions>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Roadmap & Requirements
- `.planning/ROADMAP.md` §"Phase 2: Process Lifecycle Port" (lines 34-42) — goal, success criteria, requirement IDs
- `.planning/REQUIREMENTS.md` lines 18-22 — PROC-01, PROC-02, PROC-03 definitions

No external ADRs/specs exist for this phase — requirements fully captured in ROADMAP.md/REQUIREMENTS.md and the decisions above.

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- `engine/backend/orgtree/gitrunner.py:36-48` (`_stop()`) — the canonical POSIX termination pattern to replicate: `os.name == "nt"` branch uses `taskkill /PID <pid> /T /F`; else branch uses `os.killpg(proc.pid, signal.SIGKILL)` (relies on the Popen call having been started with a new session/process group so `proc.pid` is the group leader).
- `engine/backend/orgtree/providers.py:447` (`codex_path()`) and `:883` (`antigravity_path()`) — the env-override > private-pin-under-data-root > `shutil.which()` PATH-fallback pattern already mitigates most GUI-launch PATH-restriction concerns for Codex/Antigravity; PROC-01 for those two is a verification task, not new design.
- `engine/backend/orgtree/liveness.py` — already has a POSIX branch using `os.kill(pid, 0)`; PROC-02 is a verification task against real macOS behavior (this session's own environment is Darwin — real macOS testing is directly possible here, not simulated).

### Established Patterns
- Windows-specific call sites needing a POSIX counterpart or fix, found via `taskkill`/`ctypes` grep across `engine/`:
  - `engine/backend/orgtree/antigravityrun.py:575` — `taskkill` call (POSIX branch needs same `os.killpg` fix; also verify its `Popen` at line 756 has `start_new_session=True`)
  - `engine/backend/orgtree/gitrunner.py:38` — `taskkill` call (POSIX branch already correct — the reference pattern)
  - `engine/mailhub_runtime.py:313` — `taskkill` call (verify POSIX branch/parity)
  - `engine/backend/orgtree/codexrun.py:949` — `taskkill` call; `CodexProcess.close()` and its `Popen` at codexrun.py:506 are explicitly named by PROC-03 as missing `start_new_session`
  - `engine/backend/orgtree/managed_profiles.py:78-104` — `ctypes.WinDLL`/`advapi32`/`kernel32` calls for `_create_windows_directory()` — **not in PROC-01/02/03 scope**, unrelated to process lifecycle (directory ACLs), left untouched by this phase

### Integration Points
- `engine/launch.py:352-370` (`main()`) calls `arm_process_lifetime()` unconditionally before any of Phase 2's process-lifecycle code runs — see D-03/D-04 above; this is the reason Phase 2's fixes can be planned and reviewed but not run end-to-end on a live macOS engine yet.

</code_context>

<specifics>
## Specific Ideas

No specific requirements beyond the roadmap's success criteria — open to standard POSIX approaches for the resolver/liveness/termination work.

</specifics>

<deferred>
## Deferred Ideas

- **Data-root guardian / `process_lifetime.py` POSIX port** — genuinely blocks end-to-end macOS engine startup and Phase 5's VER-01, but is out of Phase 2's PROC-01/02/03 scope (see D-04). Recommend a new roadmap phase or decimal insertion before Phase 5 is attempted.
- **`managed_profiles.py`'s `ctypes.WinDLL` directory-ACL calls** — Windows-only, but not a process-lifecycle concern; belongs to whichever phase covers filesystem/profile setup, not this one.

</deferred>

---

*Phase: 2-Process Lifecycle Port*
*Context gathered: 2026-09-17*
