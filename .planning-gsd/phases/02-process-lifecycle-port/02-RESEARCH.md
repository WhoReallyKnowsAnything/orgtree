# Phase 2: Process Lifecycle Port - Research

**Researched:** 2026-09-17
**Domain:** POSIX process spawning, liveness detection, and process-tree termination in a Python backend (macOS port of a Windows-native engine)
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

- **D-01:** No `claude_path()`-style resolver or CLI subprocess spawn for "claude" exists anywhere in `engine/` today — only `codex_path()` (providers.py:447) and `antigravity_path()` (providers.py:883) do this, both with an env-override > private-pin > `shutil.which()` PATH fallback chain. `clipin.py` is only a version-pin constant, not a resolver. User deferred this to Claude's discretion.
- **D-02 (Claude's call):** Research first (this document) to confirm whether Claude Code is actually spawned as a subprocess somewhere outside `engine/`, or handled via a non-CLI mechanism. If no such spawn exists anywhere, build a `claude_path()` resolver in `providers.py`. If it exists elsewhere, PROC-01 for Claude Code is a verification task.
  **RESOLVED BY THIS RESEARCH — see "Critical Finding 1" below: it exists, in `supervisor.py`, not `providers.py`. D-01's premise ("no such spawn exists anywhere in engine/") is corrected: the grep that produced it searched for a `*_path()`-style function name and missed the differently-named `CLAUDE` / `_claude_argv()` pair.**
- **D-03:** `launch.py:358-361` unconditionally calls `process_lifetime.arm_process_lifetime()`, which hard-raises `RuntimeError` on any non-Windows OS. Not named in PROC-01/02/03 but blocks engine startup entirely on macOS.
- **D-04 (Claude's call):** Keep out of Phase 2's committed scope. Phase 2 stays strictly PROC-01/02/03. Flag loudly: recommend the roadmap gain a new phase (e.g. `1.1` or `2.1`) titled "Data-Root Ownership Port" before Phase 5's VER-01 is attempted.
- **D-05:** `gitrunner.py:36-48`'s existing POSIX branch does an immediate `os.killpg(proc.pid, signal.SIGKILL)` with no grace period — this is the pattern the roadmap says to match.
- **D-06 (Claude's call):** `codexrun.py::CodexProcess.close()` and `antigravityrun.py`'s termination path should copy `gitrunner.py`'s exact immediate-SIGKILL behavior. Both call sites need `start_new_session=True` added to their `Popen(...)` calls first (codexrun.py confirmed missing it; antigravityrun.py's Popen at antigravityrun.py:756 needed verification).
  **RESOLVED BY THIS RESEARCH: antigravityrun.py:756 is confirmed MISSING `start_new_session=True` too — same bug as codexrun.py, not merely "needs verifying."**

### Claude's Discretion

All three areas above (D-02, D-04, D-06) were explicitly deferred to Claude ("You decide"). The planner and researcher should treat these as decided, not re-open them with the user — **except** the new scope question raised by Critical Finding 2 below (supervisor.py's own Claude-CLI spawn/kill sites), which is genuinely new information CONTEXT.md's authors did not have, and is flagged for a scope decision rather than silently absorbed.

### Deferred Ideas (OUT OF SCOPE)

- **Data-root guardian / `process_lifetime.py` POSIX port** — genuinely blocks end-to-end macOS engine startup and Phase 5's VER-01, but is out of Phase 2's PROC-01/02/03 scope (see D-04). Recommend a new roadmap phase or decimal insertion before Phase 5 is attempted.
- **`managed_profiles.py`'s `ctypes.WinDLL` directory-ACL calls** — Windows-only, but not a process-lifecycle concern; belongs to whichever phase covers filesystem/profile setup, not this one.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|------------------|
| PROC-01 | Executable resolution (`providers.py` and related) resolves provider CLIs (Claude Code, Codex, Antigravity) correctly on macOS | Confirmed all three resolvers exist and are correct in shape (env > pin/install > PATH). Codex and Antigravity resolvers live in `providers.py`; Claude's equivalent (`CLAUDE`, `claude_install_state()`, `_claude_argv()`) lives in `supervisor.py` — Critical Finding 1. Live-probed on this Darwin arm64 machine: `shutil.which("claude")` and `shutil.which("codex")` both resolve correctly today. |
| PROC-02 | Process liveness checks work on macOS — verify existing POSIX branch in `liveness.py` (`os.kill(pid, 0)`) | `liveness.py::_observe_pid` (lines 159-199) already correctly distinguishes alive / no-such-process / access-denied / probe-error. Live-probed on this machine: confirms correct behavior for live and reaped processes, and confirms a real **zombie-process gap** (a child that exited but was never `.wait()`-ed still reports alive) — see Common Pitfalls. |
| PROC-03 | Child-process containment (killing a provider CLI kills its descendants) works on macOS for every process spawn path | `gitrunner.py:36-48` confirmed correct (reference pattern). `codexrun.py` (Popen:506, close():945-959) and `antigravityrun.py` (Popen:756, kill_tree():566-582) both confirmed missing `start_new_session=True` and both terminate via bare `proc.kill()`, never `os.killpg()`. **Critical Finding 2:** `supervisor.py`'s own Claude Code CLI spawn/kill sites (the primary, most-used provider) have the identical gap and were not named in the roadmap's established-patterns list. |
</phase_requirements>

## Summary

This phase ports three narrow but load-bearing pieces of process-lifecycle code from a Windows-only implementation to a working POSIX (macOS) implementation: executable resolution, liveness checking, and process-tree termination. The codebase already has one **fully correct** reference implementation (`gitrunner.py`'s `_stop()`), one **fully correct** liveness primitive (`liveness.py::_observe_pid`, not yet wired to any provider-CLI production call site but independently tested), and two resolvers (`codex_path()`, `antigravity_path()`) that already implement the right fallback chain. The work is almost entirely "copy an already-correct pattern to two-to-four more call sites," not new design.

The single most important finding of this research is that **Claude Code's own CLI resolution already exists** — not as a missing `claude_path()` in `providers.py` (which is what CONTEXT.md's D-01/D-02 assumed after a `providers.py`-scoped grep), but as a `CLAUDE` module constant plus `_claude_argv()` function in `engine/backend/orgtree/supervisor.py`, using the identical env-override > private-pin > `shutil.which()` chain. PROC-01 for Claude Code is therefore a **verification-only** task; no new resolver code should be written.

The second, larger finding is that `supervisor.py` — the 32,746-line file that owns every real Claude Code turn — spawns the Claude CLI at four call sites (main turn, cache-keepalive, fork/compact, remote-control), **none of which set `start_new_session=True` on POSIX**, and terminates them exclusively via bare `proc.kill()` / `proc.terminate()` (in `_wd_kill_tree`, `_reap_orphans`, and inline calls), **never `os.killpg()`**. The single production-facing "stop this agent" entry point (`warmpool.halt_kill()` → `sup._wd_kill_tree()`) inherits this gap. This is materially larger in blast radius than the `codexrun.py` gap the roadmap names explicitly, because Claude is the incumbent/default provider and this is the code path a user's "stop" button actually calls. This was not in CONTEXT.md's established-patterns list — flagged loudly below (Critical Finding 2) for a scope decision, following the same "flag loudly, don't silently absorb or ignore" precedent CONTEXT.md itself set for D-04.

Live probes were run on this session's own Darwin arm64 (macOS) machine — confirming `shutil.which()` PATH resolution for `claude` and `codex`, confirming `os.kill(pid, 0)` correctly reports both live and reaped-dead processes, confirming the zombie-process edge case is real (an un-`wait()`-ed exited child still reads as alive), and confirming `os.killpg()` after `start_new_session=True` correctly kills a process group.

**Primary recommendation:** Copy `gitrunner.py`'s exact `_stop()` pattern (`start_new_session=True` on the `Popen()` call + `os.killpg(proc.pid, signal.SIGKILL)` on termination, POSIX branch only — Windows branches are untouched) to `codexrun.py` and `antigravityrun.py`. Verify (do not rebuild) Claude/Codex/Antigravity executable resolution against real macOS install paths. Bring the supervisor.py scope question (Critical Finding 2) to the planner explicitly rather than silently including or excluding it.

## Architectural Responsibility Map

This is a two-tier desktop application (Electron shell + local Python backend engine), not a client/server web app — the standard browser/SSR/API/CDN/DB tiers do not apply cleanly. Adapted tiers below.

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| Executable resolution (find the CLI binary) | Python Backend Engine (`providers.py`, `supervisor.py`) | OS filesystem / PATH | Pure Python stdlib (`shutil.which`, `os.path.exists`) reading OS-level install locations; no Electron/renderer involvement. |
| Process liveness check | Python Backend Engine (`liveness.py`) | OS process table | `os.kill(pid, 0)` is a direct OS syscall wrapper; the engine is the only layer that holds pids worth checking. |
| Process spawn (Popen) | Python Backend Engine (`supervisor.py`, `codexrun.py`, `antigravityrun.py`, `gitrunner.py`) | OS process/session groups | `start_new_session=True` is a Python `subprocess` kwarg that asks the OS kernel to create a new POSIX session/process group at fork time — this is an OS-tier decision made from the backend tier. |
| Process-tree termination | Python Backend Engine (`_wd_kill_tree`, `close()`, `kill_tree()`, `_stop()`) | OS process/session groups | `os.killpg()` operates on the OS's process-group table; the engine only supplies the pgid. |
| Electron main process | apps/desktop/main | — | Spawns the Python engine itself (`launch.py`) and relays UI events; does **not** spawn provider CLIs directly. Out of this phase's scope — no Electron-side changes needed for PROC-01/02/03. |

## Critical Finding 1: Claude Code CLI resolution already exists (supervisor.py, not providers.py)

**Location:** `engine/backend/orgtree/supervisor.py:320-334` and `:528-533`.

```python
# supervisor.py:320-326 — module load time
_DATA = os.path.expanduser(os.environ.get("ORGTREE_DATA", "~/orgtree"))
_PIN = os.path.join(_DATA, "cli", "node_modules", "@anthropic-ai",
                    "claude-code", "bin", "claude.exe" if os.name == "nt"
                    else "claude")
CLAUDE = (os.environ.get("ORGTREE_CLAUDE")
          or (_PIN if os.path.exists(_PIN) else None)
          or shutil.which("claude") or "claude")
```
```python
# supervisor.py:528-533
def _claude_argv() -> list[str]:
    if os.path.exists(CLAUDE_CLI_JS):
        return ["node", CLAUDE_CLI_JS]
    if os.name == "nt" and CLAUDE.lower().endswith((".cmd", ".bat")):
        return ["cmd", "/c", CLAUDE]
    return [CLAUDE]
```
`[VERIFIED: engine/backend/orgtree/supervisor.py:320-326,528-533 — read this session]`

There is also a runtime-refreshed presence check, `claude_install_state()` (supervisor.py:352-388), which re-probes the same env > pin > PATH order behind a 60s cache — explicitly documented as mirroring `providers.codex_status`'s trust model (env taken on faith but existence-checked; pin/PATH hits exist by construction). `[VERIFIED: engine/backend/orgtree/supervisor.py:352-388 — read this session]`

**What this means for PROC-01:** the resolution chain for all three providers is env-override > private-pin/install-location > `shutil.which()` PATH fallback, already implemented for all three. Claude's implementation is scattered across `supervisor.py` (constant + argv builder + install-state probe) rather than a single `providers.py` function, which is why the earlier `providers.py`-scoped/`*_path()`-named grep in CONTEXT.md's D-01 missed it. **No new resolver code is needed for PROC-01.** The task is: verify `shutil.which("claude")`, `codex_path()`, and `antigravity_path()` each resolve correctly against real macOS install layouts (see live-probe results below), and — if the planner wants naming consistency across the three providers — optionally note (not necessarily fix, since it's out of the "no new capabilities" phase boundary) that `providers.py` has no equivalent to `CLAUDE`/`_claude_argv()`.

## Critical Finding 2: supervisor.py's Claude-CLI spawn/kill sites share the same gap as codexrun.py — flagged for a scope decision

**This file was not named anywhere in CONTEXT.md's "Established Patterns" or "Integration Points" sections.** It is 32,746 lines; the process-lifecycle-relevant Popen/kill sites are spread across it rather than concentrated in one `close()`/`_stop()` method the way the three purpose-built runner files are, which is almost certainly why the phase's original context-gathering grep did not surface it.

**Popen call sites for the actual Claude Code CLI process — none pass `start_new_session`:**

| Site | Line | Purpose | POSIX kwargs |
|------|------|---------|--------------|
| Main real turn | supervisor.py:18321 | The normal "run a turn" spawn (`spawn_argv(org, nid, _build_cmd(org, nid))`) | `creationflags=(CREATE_NO_WINDOW if nt else 0)` only |
| Cache keepalive | supervisor.py:12340 | Background cache-warming ping | same |
| Fork/compact | supervisor.py:24011 | `/compact` session-fork spawn | same |
| Remote-control | supervisor.py:24305 | Remote-control session spawn | same |

`[VERIFIED: engine/backend/orgtree/supervisor.py:18321-18326,12340-12345,24011-24016 — read this session; exact lines quoted below]`

```python
# supervisor.py:18321-18326 (the main turn spawn)
proc = subprocess.Popen(
    spawn_argv(org, nid, _build_cmd(org, nid)),
    cwd=scratch_dir(slug, nid), env=env,
    stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE,
    text=True, encoding="utf-8", errors="replace",
    creationflags=(subprocess.CREATE_NO_WINDOW
                   if os.name == "nt" else 0))
```

**Termination sites — none use `os.killpg()`:**

- `_leash()` / `_reap_orphans()` (supervisor.py:1256-1275): on POSIX, `_leash` just adds the `Popen` handle to an `_ORPHANS` set; the `atexit`-registered `_reap_orphans()` calls `p.kill()` on each — single-process kill only.
  `[VERIFIED: engine/backend/orgtree/supervisor.py:1256-1275 — read this session]`
- `_wd_kill_tree()` (supervisor.py:29476-29505): POSIX branch is `proc.kill()` then `proc.wait(timeout=5)` — no `os.killpg` anywhere in the function, despite its own docstring stating the goal is "Kill a dog's child AND everything it started" and citing a measured Windows orphan leak (`ping` surviving 27 hours) as the reason `taskkill /T` exists. The POSIX branch never received the equivalent fix.
  `[VERIFIED: engine/backend/orgtree/supervisor.py:29476-29505 — read this session]`
- `warmpool.halt_kill()` (warmpool.py:2376-2382) — **the production "stop this agent" entry point** — calls `sup._wd_kill_tree(proc)` for every process in `_terminating`. This is the function a user's halt/stop action actually invokes.
  `[VERIFIED: engine/backend/orgtree/warmpool.py:2376-2382 — read this session]`
- Inline `proc.kill()` (supervisor.py:24026) and `proc.terminate()` (supervisor.py:24341,24385,24394) at the fork/compact and remote-control timeout paths — same single-process pattern.
  `[VERIFIED: grep of engine/backend/orgtree/supervisor.py this session — no `os.killpg` call exists anywhere in the file]`

**Why this matters:** roadmap Success Criterion #3 says "Stopping an agent kills the provider CLI **and all of its descendant processes** on macOS," worded generically across all three providers, not scoped to Codex. Claude Code is the incumbent/default provider (per `claude_install_state()`'s own docstring: "Claude was the incumbent provider"). If Claude Code — or any tool it shells out to via MCP servers or subagents — forks a child process, that child survives a "stop" on macOS today, exactly the orphan-process failure mode PROC-03 exists to close, just on the highest-traffic call path rather than the one named in the roadmap text.

**This is presented as a flagged scope question, not a silent scope change**, following the CONTEXT.md precedent set for D-04 (flag loudly, let the planner/user decide rather than either silently absorbing it into "3 files" or silently ignoring it because it wasn't in the original list). Two honest options for the planner:
1. **Expand PROC-03's task list** to include `supervisor.py`'s four Popen sites + `_wd_kill_tree`/`_reap_orphans`, using the same `start_new_session=True` + `os.killpg()` pattern. This is the literal, complete reading of the roadmap's success criterion.
2. **Descope it explicitly** with a documented rationale (e.g., "supervisor.py is too large/high-risk to touch safely in this phase; tracked as a follow-up") — mirroring how D-04 was descoped, so the gap is a decision, not an oversight.

Given `_wd_kill_tree` is shared code (used by the watchdog "dog" subsystem too, which the phase boundary explicitly does not cover), option 1's blast radius is larger than "add one kwarg to one file" — changing `_wd_kill_tree`'s POSIX branch affects every caller, not just Claude turns. A narrower version of option 1 exists: add `start_new_session=True` to the four Claude Popen sites and add a **new**, Claude-turn-specific kill helper (mirroring `gitrunner._stop`) rather than editing the shared `_wd_kill_tree`, leaving the watchdog subsystem's behavior untouched. This avoids widening the phase into "fix all of supervisor.py's process handling."

## Standard Stack

### Core

No third-party libraries are introduced by this phase. All process-lifecycle work uses Python 3 stdlib, already imported in every file involved.

| Module | Purpose | Why Standard |
|--------|---------|---------------|
| `subprocess` | `Popen`, `start_new_session` kwarg, `CREATE_NO_WINDOW`/`CREATE_NEW_PROCESS_GROUP` flags | Already used throughout the codebase; `start_new_session` has been stable since Python 3.2 |
| `os` | `os.kill(pid, 0)` liveness probe, `os.killpg(pgid, signal)` tree kill, `os.getpgid` | Direct POSIX syscall wrappers; no alternative needed |
| `signal` | `signal.SIGKILL` | Standard POSIX signal constant |
| `shutil` | `shutil.which()` | Already the PATH-fallback mechanism in both existing resolvers |

`[VERIFIED: python3 --version and behavior probed live on this session's Darwin arm64 machine, see "Live Verification" below]`

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| `os.killpg()` on a `start_new_session=True` group | `psutil.Process(pid).children(recursive=True)` + individual kills | `psutil` is a new third-party dependency this project does not currently have; `os.killpg` is stdlib-only and is the pattern `gitrunner.py` already uses correctly. Only worth introducing `psutil` if a provider CLI is known to double-fork/daemonize *out of* its process group (not observed in this codebase) — not needed for this phase. |

**Installation:** none — no new packages.

## Package Legitimacy Audit

**Not applicable.** This phase adds zero external packages (npm or pip). All changes are to existing stdlib usage in already-imported modules. No `package-legitimacy check` run was needed.

## Architecture Patterns

### System Architecture Diagram

```
User clicks "Stop" in the desktop UI
        │
        ▼
apps/desktop (Electron renderer/main) ── IPC/HTTP ──▶ Python Backend Engine (FastAPI-ish app.router)
                                                              │
                                                              ▼
                                                   warmpool.halt_kill(slug, nid)
                                                              │
                                        ┌─────────────────────┼─────────────────────┐
                                        ▼                     ▼                     ▼
                              codexrun.py::close()   antigravityrun.py::kill_tree()  sup._wd_kill_tree()
                              (Codex turns)          (Antigravity turns)             (Claude turns — via
                                        │                     │                       warmpool._terminating)
                                        ▼                     ▼                     ▼
                              [BROKEN on POSIX:      [BROKEN on POSIX:        [BROKEN on POSIX:
                               proc.kill() only,      proc.kill() only,        proc.kill() only,
                               no start_new_session]  no start_new_session]    no start_new_session]
                                        │                     │                     │
                                        └─────────────────────┴─────────────────────┘
                                                              ▼
                                              CORRECT REFERENCE PATTERN:
                                              gitrunner.py::_stop()
                                              start_new_session=True at spawn
                                              + os.killpg(pid, SIGKILL) at stop
```

### Recommended Project Structure

No new files or directories — this phase edits existing files in place:
```
engine/backend/orgtree/
├── providers.py          # verify codex_path()/antigravity_path() against real macOS paths — no new code expected
├── liveness.py            # verify _observe_pid()'s os.kill(pid,0) branch against real macOS behavior — no new code expected
├── gitrunner.py           # REFERENCE ONLY — already correct, do not touch
├── codexrun.py            # ADD start_new_session=True to Popen (line 506); change close()'s POSIX kill to os.killpg (lines 945-959)
├── antigravityrun.py      # ADD start_new_session=True to Popen (line 756); change kill_tree()'s POSIX kill to os.killpg (lines 566-582)
└── supervisor.py          # SCOPE DECISION NEEDED — see Critical Finding 2
```

### Pattern 1: POSIX process-group spawn + kill (the pattern to replicate)

**What:** Start a child in its own session (making it the process-group leader), then kill the whole group by that pgid.
**When to use:** Any Popen call whose child may itself spawn further children that must not outlive a "stop."
**Example (the exact, already-correct code to copy):**
```python
# Source: engine/backend/orgtree/gitrunner.py:36-48 — read this session
def _stop(proc: subprocess.Popen[bytes]) -> None:
    if os.name == "nt":
        subprocess.run(["taskkill", "/PID", str(proc.pid), "/T", "/F"],
                       stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL,
                       creationflags=subprocess.CREATE_NO_WINDOW, timeout=5)
    else:
        try:
            os.killpg(proc.pid, signal.SIGKILL)
        except ProcessLookupError:
            pass
    if proc.poll() is None:
        proc.kill()
    proc.wait(timeout=5)
```
The `proc.pid == pgid` precondition this relies on is satisfied only because the corresponding spawn passes `start_new_session=True` — grep confirms `gitrunner.py` sets it wherever it calls `subprocess.Popen` for a tracked child. Any file adopting `_stop()`'s kill side must also add `start_new_session=True` to its spawn side, or `os.killpg` will target the wrong (or the engine's own) process group.

### Anti-Patterns to Avoid

- **Bare `proc.kill()`/`proc.terminate()` as "tree kill":** kills only the direct child. If that child is a shell/CLI wrapper that forks its own children (both `codexrun.py`'s and `antigravityrun.py`'s own docstrings confirm their targets do this — Codex forks a native engine + code-mode-host child; Antigravity forks a language-server child), the grandchildren are orphaned. This is the exact, already-diagnosed Windows failure mode (`codexrun.py:930-937`, measured 2026-08-30) that has no POSIX-side fix yet.
- **Adding `start_new_session=True` without also changing the kill call, or vice versa:** the two must move together. `os.killpg(proc.pid, ...)` on a process that was *not* started with `start_new_session=True` targets the engine's own process group (self-inflicted, dangerous) rather than a no-op.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| Process-tree termination on POSIX | A custom `psutil`-based recursive-children walker | `start_new_session=True` + `os.killpg(pid, SIGKILL)` | stdlib-only, already the correct pattern in `gitrunner.py`; introducing `psutil` adds a new dependency for a problem the stdlib already solves for this codebase's spawn shape (no daemonizing/double-forking children observed) |
| Liveness detection | A new tri-state prober | `liveness.py::observe()`/`_observe_pid()` (already exists, already tested) | Duplicating this logic elsewhere risks the exact "two copies that can disagree" failure the liveness.py docstring itself warns against re: `supervisor._pid_provably_dead` |

**Key insight:** every piece of this phase already has a correct reference implementation somewhere in the codebase (`gitrunner.py` for kill, `liveness.py` for liveness, `providers.py` for two of three resolvers). The job is disciplined replication, not invention — and the risk is copying the pattern to the two named files while missing that four more spawn/kill sites in `supervisor.py` need the identical fix.

## Common Pitfalls

### Pitfall 1: zombie processes report as "alive"

**What goes wrong:** `os.kill(pid, 0)` succeeds (no exception) for a child that has already exited but has not yet been reaped via `.wait()`/`.poll()` — the OS keeps the pid as a zombie table entry until the parent collects its exit status.
**Why it happens:** POSIX semantics — a zombie still occupies a process-table slot; `kill(pid, 0)` only tests "does this pid exist," not "is this process actually running."
**How to avoid:** any consumer of `liveness.py::observe()`/`_observe_pid()` for a process the engine itself spawned must ensure `.poll()`/`.wait()` is called promptly on exit (the existing `Popen`-holding code paths in `codexrun.py`/`antigravityrun.py`/`gitrunner.py` do call `.wait()` after killing, which reaps in time) — this pitfall mainly bites long-lived engine-external pids (e.g. a pid persisted to disk across an engine restart, which `mailhub_runtime.py`'s `hub.pid` mechanism does — that pid is *not* one of this phase's providers, but is the same class of hazard).
**Warning signs:** a "stopped" provider process that still shows as alive in a liveness check for longer than expected — reap it explicitly rather than assuming the check is broken.
`[VERIFIED: live python3 probe on this session's Darwin arm64 machine — exact reproduction below]`

### Pitfall 2: PID reuse across a stored/persisted pid

**What goes wrong:** the OS recycles pids. A pid captured once (e.g., written to a state file) can, after the original process exits, be reassigned to an unrelated new process — `os.kill(stale_pid, 0)` then reports "alive" for the wrong process.
**Why it happens:** standard POSIX pid allocation.
**How to avoid:** none of PROC-01/02/03's targeted call sites persist a bare pid across a process lifetime for later re-identification — they hold the live `Popen` object directly (`.poll()` doesn't share this hazard, since it's implemented via `waitpid`, not by pid). The one place in this codebase that *does* persist a bare pid (`mailhub_runtime.py::_reclaim_orphan`, reading `hub.pid`) is explicitly a different subsystem (the internal mail server, not a provider CLI) and is out of this phase's scope — flagged only so the planner does not accidentally fold it into PROC-03's Popen-based fixes, which follow a different (safer) pattern.
**Warning signs:** any future code that starts checking liveness via a pid read back from disk rather than a live `Popen` handle needs its own staleness guard (e.g. comparing process start time), which `liveness.py` does not currently provide and PROC-02 does not ask for.

### Pitfall 3: `os.killpg` without `start_new_session` is a self-inflicted kill

**What goes wrong:** calling `os.killpg(proc.pid, SIGKILL)` on a child spawned *without* `start_new_session=True` sends SIGKILL to the caller's own process group (since the child inherited the parent's pgid) — this can kill the Python backend engine itself.
**Why it happens:** `proc.pid == pgid` is only true when the child was made a session/group leader at spawn time.
**How to avoid:** always add `start_new_session=True` to the `Popen()` call in the same change that adds `os.killpg()` to the corresponding kill path — never one without the other. This is exactly why `codexrun.py` and `antigravityrun.py` both need a two-part fix (Popen line **and** close/kill_tree function), not a one-line patch.
**Warning signs:** a change that only touches the kill function and not the spawn call (or vice versa) should fail review.

## Code Examples

### Live-verified macOS process-lifecycle behavior (this session, Darwin arm64, 2026-09-17)

```python
# Executed via python3 on this session's own macOS machine (Darwin 27.2.0, arm64)
import subprocess, os, time, signal

# 1. liveness check on a live child
p = subprocess.Popen(["sleep", "2"])
os.kill(p.pid, 0)                 # -> succeeds, no exception ("alive")

# 2. liveness check on a reaped (waited) dead child
p.wait()
os.kill(p.pid, 0)                 # -> raises ProcessLookupError ("correctly dead")

# 3. zombie (exited but NOT yet reaped) child
p2 = subprocess.Popen(["true"])
time.sleep(0.3)                   # exits, but .wait()/.poll() not called yet
os.kill(p2.pid, 0)                # -> STILL succeeds, no exception (zombie hazard, confirmed)
p2.wait()

# 4. process-group kill after start_new_session=True
p3 = subprocess.Popen(["/bin/sh", "-c", "sleep 30 & wait"], start_new_session=True)
pgid = os.getpgid(p3.pid)         # -> equals p3.pid (group leader)
os.killpg(pgid, signal.SIGKILL)
p3.poll()                         # -> -9 (killed by SIGKILL)
```
`[VERIFIED: executed this session via ctx_shell on Darwin 27.2.0 arm64]`

### Real macOS PATH resolution (this session, Darwin arm64, 2026-09-17)

```
$ which claude codex agy
~/.local/bin/claude
/opt/homebrew/bin/codex
agy not found
```
`[VERIFIED: shell probe this session]` — confirms `shutil.which("claude")` and `shutil.which("codex")` resolve on a real Apple Silicon macOS install today; `agy` correctly returns nothing when Antigravity isn't installed (matches `_antigravity_install_path()`'s `~/.local/bin/agy` check returning `None` when absent). Note the specific paths (`~/.local/bin` for `claude`, Homebrew's `/opt/homebrew/bin` for `codex`) are this-machine-specific install choices, not something the resolver code hardcodes — `shutil.which()` is PATH-driven and install-location-agnostic by design, so this result generalizes to any macOS machine with these CLIs on `PATH`. `[ASSUMED: generalization to other macOS machines/PATH layouts beyond this one — not independently verified on a second machine this session]`

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| Windows-only `taskkill /PID <pid> /T /F` for tree kill | POSIX `start_new_session=True` + `os.killpg(pid, SIGKILL)` | N/A — this is a platform port, not a version upgrade | `gitrunner.py` already made this transition correctly; `codexrun.py`/`antigravityrun.py` (and, per Critical Finding 2, `supervisor.py`) have not |

**Deprecated/outdated:** none — this is a first-time POSIX implementation, not a migration off a deprecated API.

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | The `~/.local/bin/claude` and Homebrew `/opt/homebrew/bin/codex` install paths seen on this one probe machine generalize to "typical" real-world macOS installs of these CLIs | Code Examples — Real macOS PATH resolution | Low: `shutil.which()`'s correctness does not depend on any specific install path, only on the CLI being somewhere on `PATH` — so this assumption affects documentation/expectation-setting only, not the resolver's actual correctness, which is PATH-agnostic by construction |
| A2 | No provider CLI in this codebase double-forks/daemonizes in a way that escapes its own process group (which would make `os.killpg` insufficient and require a `psutil`-style tree walk instead) | Don't Hand-Roll | Medium: if a future Codex/Antigravity/Claude CLI version starts a detached background helper (e.g. a long-lived MCP server it does NOT keep as a direct child of its own session), `os.killpg` would miss it; not currently observed in any read source file this session |

## Open Questions

1. **Should PROC-03's scope include `supervisor.py`'s four Claude-CLI Popen/kill sites?**
   - What we know: they have the identical `start_new_session`/`os.killpg` gap as `codexrun.py`, and the production "stop" path (`warmpool.halt_kill`) routes through them.
   - What's unclear: whether the phase's "no new capabilities, PROC-01/02/03 only" boundary was meant to exclude `supervisor.py` deliberately (unlikely, since CONTEXT.md's authors did not know about this file's gap) or simply didn't consider it.
   - Recommendation: surface this to the planner as a go/no-go decision before task breakdown, using the narrower fix option described in Critical Finding 2 (new Claude-turn-specific kill helper, not editing the shared `_wd_kill_tree`) if the decision is "yes, include it."

2. **Does the codebase want `providers.py` to gain a `claude_path()`-shaped wrapper for naming consistency, even though `supervisor.py`'s existing resolution already works?**
   - What we know: functionally nothing is broken; `CLAUDE`/`_claude_argv()` already do the job.
   - What's unclear: whether "no new capabilities" also means "no new pure-refactor code," which would rule this out anyway.
   - Recommendation: skip it — the phase boundary says no new capabilities, and this would be a rename/consistency change, not a fix.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| Python 3 (stdlib `subprocess`/`os`/`signal`/`shutil`) | All of PROC-01/02/03 | ✓ | 3.13 target per `tools/provision-runtime.py:45` (`--python-version 3.13`); `start_new_session` stable since 3.2 | — |
| `claude` CLI on PATH | PROC-01 verification | ✓ (this machine) | `~/.local/bin/claude` | — |
| `codex` CLI on PATH | PROC-01 verification | ✓ (this machine) | `/opt/homebrew/bin/codex` | — |
| `agy` (Antigravity) CLI on PATH | PROC-01 verification | ✗ (this machine) | not installed | Verification of `antigravity_path()`'s `~/.local/bin/agy` branch requires either installing Antigravity on a test machine, or a mocked-path unit test (already the pattern `test_codex_home_usage.py` uses for `codex_path`) |

**Missing dependencies with no fallback:** none blocking.
**Missing dependencies with fallback:** Antigravity CLI itself is not installed on this development machine; PROC-01's Antigravity resolution can still be verified via a mocked-path test plus manual confirmation on a machine that has it installed.

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Python `unittest` (stdlib) — no pytest.ini/pyproject.toml/conftest.py found; confirmed via `docs/known-failures.md:246`, `docs/scope-diagnostics.md:24` |
| Config file | none — see Wave 0 |
| Quick run command | `python -m unittest tests.test_provider_attempt_and_liveness -v` |
| Full suite command | `python -m unittest discover -s tests -p 'test_*.py'` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PROC-01 | `codex_path()`/`antigravity_path()` resolve correctly | unit (mocked path) | `python -m unittest tests.test_codex_home_usage -v` | ✅ (mocks `codex_path`; no equivalent for `antigravity_path` or Claude's `CLAUDE`/`_claude_argv` — Wave 0) |
| PROC-02 | `os.kill(pid, 0)` correctly distinguishes alive/dead/unknown, including zombie/reaped edge cases | unit (real spawn+reap) | `python -m unittest tests.test_provider_attempt_and_liveness -v` | ✅ `test_3e_a_live_process_is_alive_and_a_reaped_one_is_dead` already covers this exact behavior |
| PROC-03 | Killing a provider CLI kills its descendants on POSIX | integration (real process tree) | none exists today | ❌ Wave 0 — every existing halt/kill test (`test_agent_halt.py`, `test_claude_pipe_lifecycle.py`, `test_org_killswitch.py`) **mocks `_wd_kill_tree` entirely** via `patch.object(sup, "_wd_kill_tree", ...)`, so none of them actually exercises real descendant-process reaping |

### Sampling Rate

- **Per task commit:** `python -m unittest tests.test_provider_attempt_and_liveness -v` (fast, covers PROC-02's core logic)
- **Per wave merge:** `python -m unittest discover -s tests -p 'test_*.py'`
- **Phase gate:** full suite green, plus a new real-process-tree integration test (Wave 0 gap below) before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] A real-process-tree termination test for `codexrun.py::CodexProcess.close()` — spawn a shell that itself forks a child (mirroring the `sleep 30 & wait` shape used in this session's live probe), call `close()`, assert the grandchild is also gone. No such test exists; every current test mocks `_wd_kill_tree`/the kill call entirely.
- [ ] The same for `antigravityrun.py::kill_tree()`.
- [ ] If Critical Finding 2's scope question resolves "yes, include supervisor.py": the same real-process-tree test for whichever new Claude-turn kill helper is added.
- [ ] A mocked-path resolution test for `antigravity_path()` and for `supervisor.CLAUDE`/`_claude_argv()`, mirroring `test_codex_home_usage.py`'s existing pattern for `codex_path()` — none currently exists for the other two providers.

*(Framework itself is present and sufficient — `unittest` needs no new install.)*

## Security Domain

This phase is backend process-management code in a local desktop engine, not a network-facing authentication/session surface — most ASVS web-application categories (V2 Authentication, V3 Session Management, V4 Access Control) do not apply to the code being changed here.

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-----------------|
| V2 Authentication | No | Not touched by this phase |
| V3 Session Management | No | Not touched by this phase |
| V4 Access Control | No | Not touched by this phase |
| V5 Input Validation | No | No new user input is parsed by this phase's changes (executable paths come from `shutil.which()`/fixed install-location checks, not user-supplied strings) |
| V6 Cryptography | No | Not touched by this phase |

### Known Threat Patterns for this domain

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|---------------------|
| Orphaned provider-CLI descendant process retains injected credentials (`env = spawn_env(...)`, which carries the resolved account's auth) after the user believes the agent is "stopped" | Information Disclosure / Denial of Service (resource exhaustion) | `os.killpg()` after `start_new_session=True`, ensuring the entire credential-holding process tree terminates atomically with the parent — this is the actual security value PROC-03 delivers, framed correctly: not an ASVS web-category fix, but closing a live-credential-retention window on process stop |
| Stale/persisted pid reused by an unrelated process, misidentified as the original provider CLI (see Pitfall 2) | Spoofing (of process identity) | Prefer holding the live `Popen` object over persisting bare pids across restarts; the phase's three targeted call sites already do this correctly (they hold `Popen` handles, not disk-persisted pids) |

## Sources

### Primary (HIGH confidence — direct codebase reads this session)
- `engine/backend/orgtree/gitrunner.py:1-55` — canonical `_stop()` pattern
- `engine/backend/orgtree/liveness.py:1-215` — tri-state liveness model, `_observe_pid`
- `engine/backend/orgtree/providers.py:384-460,860-905` — `codex_path()`, `antigravity_path()`, argv builders
- `engine/backend/orgtree/supervisor.py` (multiple ranges: 320-650, 1200-1290, 10139-10175,10260-10280,10525-10570, 12300-12360, 18295-18335, 23985-24030, 26690-26760, 29270-29300, 29417-29530) — `CLAUDE`, `_claude_argv`, `claude_install_state`, `_leash`/`_reap_orphans`, `_wd_kill_tree`, `_pid_provably_dead`, all Claude-CLI Popen sites
- `engine/backend/orgtree/codexrun.py:480-515,930-965` — Popen init, `close()`
- `engine/backend/orgtree/antigravityrun.py:555-595,740-770` — `kill_tree()`, `launch()`
- `engine/mailhub_runtime.py:290-385` — `_reclaim_orphan()`, `start()` (confirmed out-of-scope subsystem)
- `engine/launch.py:340-372`, `engine/process_lifetime.py:1-50` — D-03/D-04's cited blocker, confirmed
- `engine/backend/orgtree/clipin.py` — confirmed version-pin-only, no resolver
- `engine/backend/orgtree/warmpool.py:2360-2420` — `halt_kill()`, the production stop entry point
- `tests/test_provider_attempt_and_liveness.py`, `tests/test_agent_halt.py`, `tests/test_claude_pipe_lifecycle.py`, `tests/test_org_killswitch.py`, `tests/test_codex_home_usage.py`, `tests/restart_kill_probe.py` — existing test coverage audit
- Live shell probes executed this session on Darwin 27.2.0 arm64 (this machine): `which claude codex agy`, and a Python script exercising `os.kill(pid,0)` (live/reaped/zombie) and `os.killpg` after `start_new_session=True`

### Secondary (MEDIUM confidence)
- None used — no external documentation lookups were needed; this phase's questions are entirely about this repository's own code and long-stable Python stdlib behavior, both verified directly.

### Tertiary (LOW confidence)
- None.

## Metadata

**Confidence breakdown:**
- Standard stack (stdlib-only, no new deps): HIGH — verified via direct source reads and live execution on the actual target OS
- Architecture / existing patterns: HIGH — every cited pattern was read from the file this session, with exact line ranges
- Critical Finding 1 (Claude resolution exists in supervisor.py): HIGH — read directly, function names and line numbers confirmed
- Critical Finding 2 (supervisor.py Claude-CLI kill gap): HIGH on the facts (grep + direct reads confirm no `os.killpg` anywhere in the file); the *scope recommendation* is a judgment call flagged explicitly for the planner, not asserted as decided
- Pitfalls (zombie process, pid reuse, killpg-without-start_new_session): HIGH — zombie and killpg behavior independently reproduced via live execution this session on the actual target platform (Darwin arm64)

**Research date:** 2026-09-17
**Valid until:** 30 days (stable domain — POSIX process semantics and this codebase's own file layout do not change on a fast cadence; re-verify if `supervisor.py`'s Claude-spawn code is refactored before this phase executes)
