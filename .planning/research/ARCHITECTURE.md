# Architecture Patterns

**Domain:** Cross-platform subprocess lifecycle management (Electron + Python engine spawning CLI agent processes)
**Researched:** 2026-09-17
**Confidence:** HIGH (Python stdlib docs + direct verification against existing codebase patterns already in `engine/backend/orgtree/`)

## Recommended Architecture

The Windows-only code in this codebase does three distinct jobs, each with a direct POSIX/macOS equivalent. None require a new dependency for the core mechanism; `psutil` is already an approved backend dependency (`frozen_install.py`) and should be reused, not re-justified, where a richer process view is needed.

| Windows mechanism | Job | macOS/POSIX equivalent | stdlib module |
|---|---|---|---|
| `ctypes.windll.kernel32.OpenProcess` / `GetExitCodeProcess` | "Is pid N alive?" | `os.kill(pid, 0)` (existence probe) or `psutil.pid_exists(pid)` / `psutil.Process(pid)` (richer state) | `os`, `psutil` |
| `taskkill /PID N /T /F` | "Kill this process and everything it forked" | Spawn into its own process group (`start_new_session=True`), then `os.killpg(pgid, signal.SIGKILL)` | `os`, `signal`, `subprocess` |
| `cmd.exe /c foo.cmd` / hardcoded `.exe`/`.cmd` suffix | "Resolve and invoke a named CLI tool" | `shutil.which("name")`, no shell wrapper, direct exec | `shutil` |

### Component Boundaries

| Component | Responsibility | Windows-only logic today | POSIX equivalent |
|---|---|---|---|
| `liveness.py::_observe_pid` | tri-state (alive/dead/unknown) pid probe used by the supervisor's remote-control detach guard | `os.name == "nt"` branch calls `ctypes.windll.kernel32.OpenProcess`/`GetExitCodeProcess`/`CloseHandle` (lines 163-189) | **Already has a correct POSIX branch** (lines 190-202): `os.kill(pid, 0)`, catching `ProcessLookupError`→no-such-process, `PermissionError`→access-denied (exists, not ours), `OSError`→probe-error. No change needed — verify it runs under macOS CI, don't rewrite it. |
| `antigravityrun.py::_short_path` | Windows 8.3 short-path alias lookup for a permission-hook path | `ctypes.windll.kernel32.GetShortPathNameW` (line 339), gated by `if os.name != "nt": return ""` (line 334) | **Already correct** — the function is a no-op on darwin, no 8.3 concept on POSIX. Nothing to port. |
| `gitrunner.py::_stop` | force-kill the `git` subprocess spawned by `run()` | `taskkill /PID … /T /F` on `os.name == "nt"` (lines 37-40) | **Already correct and the reference pattern for this whole port**: POSIX branch does `os.killpg(proc.pid, signal.SIGKILL)` (line 43), and `run()`'s `Popen` call sets `start_new_session=os.name != "nt"` (line 72) so the child gets its own process group to kill. Copy this exact shape into every other spawner. |
| `codexrun.py::CodexProcess.close()` | tear down the `codex app-server` node process and its forked native engine/host children | `taskkill /T /F /PID` (lines 946-953), then `self.proc.kill()` as a "belt" on every platform (line 955) | **Gap.** The `Popen` call in `__init__` (lines 506-511) never sets `start_new_session`/`process_group`, so on POSIX the child stays in the engine's own process group. `self.proc.kill()` only kills the node parent — the forked native engine child and `codex-code-mode-host` child are orphaned exactly the way the module's own docstring says Windows orphans them without `taskkill /T`. Fix: add `start_new_session=True` to the `Popen` call, then in `close()` add a POSIX branch calling `os.killpg(self.proc.pid, signal.SIGKILL)` before/alongside `self.proc.kill()`. |
| `providers.py` (`codex_path`, `_antigravity_install_path`, `antigravity_path`) | resolve the installed CLI binary for codex/antigravity | `.exe` suffix + `.cmd` shim path branches, `os.name == "nt"` guards (lines 242, 438, 443, 479-480, 874-877) | **Mostly already correct** — every resolver already ends in a `shutil.which("codex")` / `shutil.which("agy")` PATH fallback (lines 455, 892) for the non-pinned case. The `.exe`/`.cmd` literals are inert on POSIX since they're inside `os.name == "nt"` conditionals. Only real risk: the macOS-side npm/pin glob pattern (`_codex_pin`, lines 433-444) and the `agy` mac install path (`_antigravity_install_path`) need their actual macOS install layout confirmed (npm installs a POSIX `bin/codex` shim, Antigravity's mac install path is not `%LOCALAPPDATA%`) — a data problem, not an architecture problem. |

### Data Flow

Electron main (`apps/desktop/main/`) never talks to the agent CLIs directly — it goes through the Python engine's HTTP+bearer-token loopback API, which is already platform-neutral (per `PROJECT.md`). The three POSIX-porting concerns above are entirely inside the Python engine process (`engine/backend/orgtree/`):

```
Electron main (unchanged, platform-neutral HTTP client)
        │ HTTP + bearer token, loopback
        ▼
Python engine (engine/backend/orgtree/)
        │ subprocess.Popen(argv, start_new_session=True)  ← the fix, per spawner
        ▼
Agent CLI process (claude / codex / agy / git)
        │ may fork further children (codex app-server does)
        ▼
Grandchildren inherit the SAME process group (because start_new_session
was set on the direct child, not per-grandchild — POSIX process groups are
inherited down the fork tree unless a descendant calls setsid()/setpgid()
itself, which agent CLIs don't)
```

`os.killpg(pgid, SIGKILL)` sent to the direct child's pid (which is also the pgid, since it's the process-group leader) reaches every descendant in one syscall — this is the direct structural equivalent of what `taskkill /T` walks the process tree to do on Windows, and what a Windows Job Object does automatically for anything assigned to it.

## Patterns to Follow

### Pattern 1: Process-group containment at spawn time
**What:** Every `subprocess.Popen` call that spawns an agent/tool CLI (not just short-lived probes) sets `start_new_session=True` on POSIX, mirroring `subprocess.CREATE_NEW_PROCESS_GROUP`-equivalent Windows job-object semantics.
**When:** Any spawn of a CLI that may fork children (codex app-server, antigravity CLI, git) — i.e. anything torn down later with a force-kill, not one-shot version probes that exit on their own.
**Example (already in `gitrunner.py`, extend to `codexrun.py`/`antigravityrun.py`):**
```python
proc = subprocess.Popen(
    argv, cwd=cwd, env=env,
    creationflags=subprocess.CREATE_NO_WINDOW if os.name == "nt" else 0,
    start_new_session=os.name != "nt",
)
```
On Python 3.11+, `process_group=0` is the documented replacement for `preexec_fn=os.setpgid(0, 0)` and is thread-safe where `preexec_fn` is not — prefer it over `preexec_fn` if a distinct-from-session process group is ever needed, but `start_new_session=True` (setsid) is sufficient and already the codebase's established idiom.

### Pattern 2: Group-kill mirrors `taskkill /T /F`
**What:** Teardown sends the kill to the negative pid (the process group), not the single tracked pid.
**When:** Any `close()`/`_stop()`/`forceKillTree`-equivalent that currently does a bare `proc.kill()` on POSIX.
**Example (already in `gitrunner.py`, needs adding to `codexrun.py::close()`):**
```python
if os.name == "nt":
    subprocess.run(["taskkill", "/PID", str(proc.pid), "/T", "/F"], ...)
else:
    try:
        os.killpg(proc.pid, signal.SIGKILL)
    except ProcessLookupError:
        pass
if proc.poll() is None:
    proc.kill()   # belt-and-suspenders on the direct child either way
proc.wait(timeout=5)
```
`os.killpg` requires the pid to actually be a process-group leader (i.e. `start_new_session=True` was set at spawn) — Pattern 1 and Pattern 2 must be applied together per spawner, never one without the other, or `killpg` either raises `PermissionError`/`ProcessLookupError` or kills an unintended group.

### Pattern 3: Tri-state liveness probe, not a boolean
**What:** A pid/port probe returns a reason string (`"still-active"` / `"no-such-process"` / `"access-denied"` / `"probe-error"`), and "provably dead" is a derived predicate over that reason — never `try/except` collapsed straight to `True`/`False`.
**When:** Any code deciding whether a remote/child process is gone enough to authorize destructive recovery (this codebase's `liveness.py` already does this correctly and documents *why* in its own module docstring — a Windows `OpenProcess`-returns-0-for-both-"gone"-and-"access-denied" bug that caused a live session teardown).
**Example (already correct in `liveness.py`, the POSIX branch to keep as-is):**
```python
try:
    os.kill(pid, 0)
    return "still-active"
except ProcessLookupError:
    return "no-such-process"
except PermissionError:
    return "access-denied"          # exists, not ours — NOT dead
except OSError:
    return "probe-error"
```
`os.kill(pid, 0)` sends signal 0, which the kernel validates without actually delivering — the standard POSIX no-op liveness check. Use `psutil.pid_exists(pid)` / `psutil.Process(pid).is_running()` instead only where richer state (cmdline, listening sockets, cross-checking a pid against a port) is needed — `frozen_install.py`'s `_live_launch_inventory` already does exactly this for engine-instance detection, so it is a proven, in-repo pattern, not a new dependency to justify.

### Pattern 4: PATH resolution without a shell
**What:** Resolve a named CLI with `shutil.which(name)`, invoke the resolved path directly in `argv[0]` — no `cmd /c` / no `shell=True`.
**When:** Any "find the installed CLI" resolver (`codex_path`, `antigravity_path`, and any future provider added to `providers.py`).
**Example (already the fallback tier in `providers.py::codex_path`/`antigravity_path`):**
```python
onpath = shutil.which("codex")   # walks $PATH, checks executable bit (POSIX) or PATHEXT (Windows)
```
`shutil.which` is the direct cross-platform equivalent of Windows PATH+PATHEXT resolution: on POSIX it checks `os.X_OK` against each `$PATH` entry; on Windows it additionally tries `PATHEXT` suffixes (`.exe`, `.cmd`, `.bat`, …). Callers do not need an `os.name` branch around the *call* itself — only around suffix literals used for pinned/vendored paths that bypass `which` (e.g. `_codex_pin`'s `codex.exe` vs `codex`).

## Anti-Patterns to Avoid

### Anti-Pattern 1: Killing the direct child only, on a CLI known to fork
**What:** `self.proc.kill()` / `os.kill(pid, SIGKILL)` on just the tracked pid when the spawned CLI forks worker/engine children (codex app-server's native engine + code-mode-host children, per `codexrun.py`'s own docstring).
**Why bad:** Orphans grandchildren that hold file locks (`~/.codex` writer lock) or keep loopback ports bound; the *next* invocation fails or the engine leaks processes silently — exactly the bug class `codexrun.py`'s close() docstring already describes happening on Windows without `/T`, and Windows-without-`/T` is architecturally identical to POSIX-without-`killpg`.
**Instead:** Pattern 1 + Pattern 2 together — spawn into a new session, kill the group.

### Anti-Pattern 2: `os.killpg` without `start_new_session` at spawn
**What:** Adding `os.killpg(proc.pid, SIGKILL)` to a teardown path whose `Popen` call was never given `start_new_session=True`.
**Why bad:** The child's pgid is the *engine's own* process group (inherited from the parent by default). `killpg` on that group kills the Python engine process itself — a much worse outcome than the orphan-leak it was meant to fix.
**Instead:** Always audit the matching `Popen` call first; add `start_new_session=True` before adding any `killpg`/`-pid` kill in the same file.

### Anti-Pattern 3: Boolean liveness (`try: os.kill(pid,0); return True except: return False`)
**What:** Collapsing the probe to two states.
**Why bad:** `PermissionError` (process exists, owned by another user/sandbox) and `ProcessLookupError` (process genuinely gone) both fall into `except`, producing the exact ambiguity `liveness.py`'s own docstring documents as the root cause of a Windows remote-session teardown bug. The same collapse is possible on POSIX and would reintroduce the same class of bug on macOS.
**Instead:** Pattern 3's tri-state reason string, feeding a `dead()` predicate that only returns true for the unambiguous case.

### Anti-Pattern 4: Wrapping a `.sh`/binary CLI in a shell (`shell=True`, or `["/bin/sh", "-c", ...]`)
**What:** Porting the Windows `["cmd", "/c", exe]` wrapper literally to a POSIX `["sh", "-c", exe]` wrapper "for symmetry."
**Why bad:** POSIX executables (with a shebang line and the executable bit set) run directly; wrapping in a shell adds a quoting/injection surface (the exact class of bug `antigravityrun.py`'s own comments describe fighting on the `cmd.exe` side — `\"…\\orgtree-rights.cmd\"` argv-escaping bugs) for zero benefit, and breaks signal delivery/process-group semantics for Pattern 1/2 (the shell becomes the process-group leader, not the actual tool).
**Instead:** `shutil.which` → direct `argv[0]` = resolved path, no shell, ever, on POSIX. `antigravityrun.py::install_steering` already does this correctly for its POSIX branch (`shlex.quote` used only for building a `.sh` *script's contents*, not for wrapping a live `Popen` call in a shell).

## Scalability Considerations

Not applicable in the usual sense (this is a desktop app, not a multi-tenant service), but the process-count dimension matters directly for this port:

| Concern | 1 agent running | Several agents running concurrently | Long-running session (hours) |
|---|---|---|---|
| Process-group leak if Pattern 1/2 skipped | one orphaned codex-engine child per crash/kill | orphans multiply per-agent, each holding its own lock/port | accumulates across the session; `frozen_install.py`'s `_live_launch_inventory` (psutil-based) is the existing tool to *detect* this, not prevent it |
| Liveness probe cost | negligible | `os.kill(pid, 0)` is O(1) syscall per check; fine at any process count this app will reach | no degradation — it's a syscall, not a scan |
| PATH resolution cost | `shutil.which` does a fresh `$PATH` walk per call | cache the resolved path per provider per session if called in a hot loop (not currently a hot path here) | irrelevant at this scale |

## Sources

- [Python subprocess documentation — `start_new_session`, `process_group`](https://docs.python.org/3/library/subprocess.html) — confirms `start_new_session=True` is the current, thread-safe, documented replacement for `preexec_fn=os.setsid()`, and `process_group=<pgid>` replaces `preexec_fn=os.setpgid()` on 3.11+. (HIGH — official language documentation)
- `os.kill`, `os.killpg`, `signal.SIGKILL`/`SIGTERM` — Python stdlib `os`/`signal` module reference (HIGH — official language documentation; behavior also directly verified against this codebase's own working `gitrunner.py` implementation, which already uses this exact pair in production)
- `shutil.which` — Python stdlib reference; POSIX executable-bit vs Windows `PATHEXT` resolution behavior (HIGH — official language documentation)
- Direct source inspection of `engine/backend/orgtree/{liveness.py, gitrunner.py, codexrun.py, providers.py, antigravityrun.py, frozen_install.py}` in this repo (2026-09-17) — used to identify which Windows-specific branches already have a correct POSIX counterpart (`liveness.py`, `gitrunner.py`, `antigravityrun.py::_short_path`, most of `providers.py`) versus which have a real gap (`codexrun.py::CodexProcess.close()` missing process-group containment). (HIGH — primary source, the actual code under port)
- `.planning/PROJECT.md`, `.planning/codebase/CONCERNS.md` (this repo) — MAC-04/05/06 requirement scope and the parallel Electron-side (`apps/desktop/main/engine.ts`, `providerlogin.ts`) finding of the identical gap (no process-group kill on the TS side either), confirming this is a repo-wide pattern to fix consistently, not a single file. (HIGH — primary source)
