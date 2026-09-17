# Phase 2: Process Lifecycle Port - Pattern Map

**Mapped:** 2026-09-17
**Files analyzed:** 5 modified (no new files — all changes are in-place fixes to existing modules)
**Analogs found:** 5 / 5

## File Classification

| Modified File | Role | Data Flow | Closest Analog | Match Quality |
|----------------|------|-----------|-----------------|---------------|
| `engine/backend/orgtree/codexrun.py` (`Popen` @L506, `close()` @L926) | service (process wrapper) | event-driven (subprocess spawn/kill) | `engine/backend/orgtree/gitrunner.py` (`run()`/`_stop()`) | exact — same spawn+killpg shape |
| `engine/backend/orgtree/antigravityrun.py` (`Popen` @L756, `kill_tree()` @L567) | service (process wrapper) | event-driven | `engine/backend/orgtree/gitrunner.py` | exact |
| `engine/mailhub_runtime.py` (`Popen` @L366, `_reclaim_orphan()` @L300-320) | service (process wrapper) | event-driven | `engine/backend/orgtree/gitrunner.py` | role-match (orphan-reclaim is a different code path than live-proc kill, but same `os.name=="nt"`/POSIX split) |
| `engine/backend/orgtree/supervisor.py` (4 `Popen` sites @L18321, L12340, L24011, L24305; `_wd_kill_tree()` @L29476; `_leash`/`_reap_orphans` @L1256-1278) | service (process wrapper, highest-traffic) | event-driven | `engine/backend/orgtree/gitrunner.py` | role-match — largest surface, most sites to patch |
| `engine/backend/orgtree/providers.py` (verification only — `codex_path()` @L447, `antigravity_path()` @L883 already correct; Claude's resolver lives in `supervisor.py` as `CLAUDE`/`_claude_argv()` @L528, not here) | utility (resolver) | request-response | itself (self-referential — Codex/Antigravity are each other's analog) | exact — no new resolver needed per D-02 |
| `engine/backend/orgtree/liveness.py` (`_observe_pid()` @L159-199 — verification only, no code change expected unless zombie gap is addressed) | utility | request-response | itself | exact |

No new files are created in this phase — the roadmap scope is fixing existing spawn/kill call sites to match one already-correct reference pattern (`gitrunner.py`).

## Pattern Assignments

### `engine/backend/orgtree/codexrun.py` — PROC-03

**Analog:** `engine/backend/orgtree/gitrunner.py` `_stop()` (L36-48) and `run()`'s `Popen` call (L67-72)

**Reference termination pattern to copy** (gitrunner.py:36-48):
```python
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

**Reference spawn pattern to copy** (gitrunner.py:69-72):
```python
proc = subprocess.Popen(argv, cwd=cwd, env=env, shell=False,
                        stdin=subprocess.DEVNULL, stdout=out, stderr=err,
                        creationflags=subprocess.CREATE_NO_WINDOW if os.name == "nt" else 0,
                        start_new_session=os.name != "nt")
```

**Current broken code — `Popen` missing `start_new_session`** (codexrun.py:506-511):
```python
self.proc = subprocess.Popen(
    argv_head + list(config_overrides or []) + ["app-server"],
    stdin=subprocess.PIPE,
    stdout=subprocess.PIPE, stderr=subprocess.PIPE, env=env, cwd=cwd,
    creationflags=(subprocess.CREATE_NO_WINDOW
                   if os.name == "nt" else 0))
```
Fix: append `, start_new_session=(os.name != "nt")` to the call.

**Current broken code — `close()` uses bare `proc.kill()`, never `os.killpg()`** (codexrun.py:946-961):
```python
if os.name == "nt":
    try:
        subprocess.run(
            ["taskkill", "/T", "/F", "/PID", str(self.proc.pid)],
            check=False, capture_output=True, timeout=10,
            creationflags=getattr(subprocess, "CREATE_NO_WINDOW", 0))
    except (OSError, subprocess.SubprocessError):
        pass
try:
    self.proc.kill()          # POSIX, and a belt over taskkill
except OSError:
    pass
try:
    self.proc.wait(timeout=5)
except (subprocess.TimeoutExpired, OSError, ValueError):
    pass
```
Fix: replace the POSIX branch (`self.proc.kill()`) with `os.killpg(self.proc.pid, signal.SIGKILL)` wrapped in `try/except ProcessLookupError: pass`, matching D-06's immediate-SIGKILL (no grace period) decision — keep the existing `nt` branch and the final `.wait()` untouched. Preserve the docstring's existing rationale (tree-kill for lock contention) — it still applies, just swap the kill call.

---

### `engine/backend/orgtree/antigravityrun.py` — PROC-03

**Analog:** same as codexrun.py (`gitrunner.py`)

**Current — `Popen` missing `start_new_session`** (antigravityrun.py:756-759):
```python
self.proc = subprocess.Popen(
    self.argv, stdin=subprocess.PIPE, stdout=subprocess.PIPE,
    stderr=subprocess.PIPE, env=env, cwd=self.cwd,
    creationflags=(subprocess.CREATE_NO_WINDOW if os.name == "nt" else 0))
```
Fix: add `start_new_session=(os.name != "nt")`.

**Current — `kill_tree()` uses bare `proc.kill()`** (antigravityrun.py:567-584):
```python
def kill_tree(proc: subprocess.Popen[bytes] | None) -> None:
    if proc is None or proc.poll() is not None:
        return
    try:
        if os.name == "nt":
            subprocess.run(["taskkill", "/T", "/F", "/PID", str(proc.pid)],
                           capture_output=True, timeout=15,
                           creationflags=subprocess.CREATE_NO_WINDOW)
        proc.kill()
    except (OSError, subprocess.TimeoutExpired):
        pass
    try:
        proc.wait(timeout=15)
    except (OSError, subprocess.TimeoutExpired):
        pass
```
Fix: on POSIX, replace `proc.kill()` (currently shared between both branches) with an `os.killpg(proc.pid, signal.SIGKILL)` call scoped to the `else` (POSIX) branch — the `nt` branch's `taskkill /T` already stays. Keep the existing `except (OSError, subprocess.TimeoutExpired)` catch; add `ProcessLookupError` if not already covered by `OSError` (it is, since `ProcessLookupError` subclasses `OSError`).

---

### `engine/mailhub_runtime.py` — PROC-03

**Analog:** `gitrunner.py` (same pattern), applied to `_reclaim_orphan()`, a different code path (killing a stale pid found via a pidfile, not a held `Popen` handle).

**Current** (mailhub_runtime.py:300-324):
```python
def _reclaim_orphan(self) -> None:
    pid_path = self.data_dir / "hub.pid"
    try:
        pid = int(pid_path.read_text(encoding="utf-8").strip() or 0)
    except (OSError, ValueError):
        return
    if pid <= 0 or pid == os.getpid():
        return
    try:
        if os.name == "nt":
            subprocess.run(["taskkill", "/PID", str(pid), "/T", "/F"],
                           capture_output=True, timeout=15,
                           creationflags=getattr(
                               subprocess, "CREATE_NO_WINDOW", 0))
        else:
            os.kill(pid, 15)
    except OSError:
        pass
    try:
        pid_path.unlink()
    except OSError:
        pass
```
Note: `os.kill(pid, 15)` (SIGTERM) here kills only the single process, not a group — because this is reclaiming from a pidfile of a PAST engine run, there is no live `Popen` handle to have started with `start_new_session`, so `os.killpg()` cannot be assumed safe (the pid may not be a group leader if the prior engine run predates this phase's fix). Verify against `start()` (mailhub_runtime.py:335-366) whether the child's `Popen` call needs `start_new_session=True` added first — if yes, change `os.kill(pid, 15)` to `os.killpg(pid, signal.SIGKILL)` wrapped in `try/except ProcessLookupError` to match D-06; if the spawn side isn't in scope this session, leave `_reclaim_orphan` as SIGTERM-single-process (least risky, matches existing behavior) and flag it as a residual gap in the plan.

---

### `engine/backend/orgtree/supervisor.py` — PROC-01 (verify) + PROC-03 (fix), expanded scope per D-07

**Analog:** `gitrunner.py` for the kill pattern; `providers.py`'s `codex_path()` for the resolver pattern shape (verification only — Claude's actual resolver `_claude_argv()`/`CLAUDE` already exists here, at L528-533, confirmed correct, no change needed for PROC-01).

**4 spawn sites needing `start_new_session=True` added:**

1. Main turn spawn (supervisor.py:18321-18327):
```python
proc = subprocess.Popen(
    spawn_argv(org, nid, _build_cmd(org, nid)),
    cwd=scratch_dir(slug, nid), env=env,
    stdin=subprocess.PIPE, stdout=subprocess.PIPE, stderr=subprocess.PIPE,
    text=True, encoding="utf-8", errors="replace",
    creationflags=(subprocess.CREATE_NO_WINDOW
                   if os.name == "nt" else 0))
```
2. Cache keepalive spawn (supervisor.py:12340-12345) — identical shape.
3. Compact/fork spawn (supervisor.py:24011-24017) — identical shape.
4. Remote-control spawn (supervisor.py:24305-24310) — identical shape, uses `_claude_argv()` directly.

Fix for all 4: append `, start_new_session=(os.name != "nt")` to each `Popen(...)` call.

**Termination — `_wd_kill_tree()` (supervisor.py:29476-29507), the single chokepoint used by all 4 spawn sites' callers (via `warmpool.py` and `halt.py`) — this is the one function to patch, not each call site:**
```python
def _wd_kill_tree(proc: "subprocess.Popen[str] | None") -> None:
    if proc is None or proc.poll() is not None:
        return
    if os.name == "nt":
        try:
            subprocess.run(["taskkill", "/T", "/F", "/PID", str(proc.pid)],
                           capture_output=True, timeout=15,
                           creationflags=subprocess.CREATE_NO_WINDOW)
        except (OSError, subprocess.SubprocessError):
            pass
    try:
        proc.kill()
    except OSError:
        pass
    try:
        proc.wait(timeout=5)
    except (OSError, subprocess.TimeoutExpired):
        pass
```
Fix: replace the unconditional `proc.kill()` with an `if os.name == "nt": ...taskkill... else: os.killpg(proc.pid, signal.SIGKILL)` split (matching gitrunner.py's `_stop()` shape exactly), keeping the existing docstring's Windows-tree-kill rationale for the `nt` branch untouched, and keeping the final `.wait(timeout=5)` as a shared fallback after either branch. Since `_wd_kill_tree` is called from 6+ sites (`warmpool.py:1844,2071,2131,2382`, `halt.py:437`, and 2 more found via `_wd_kill_tree` grep), **this single-function fix propagates the SIGKILL-via-killpg pattern everywhere without touching any of those call sites** — the lazy, root-cause fix per the ponytail bug-fix rule (fix once at the shared function, not at each caller).

**Also required:** the `_leash()`/`_reap_orphans()` POSIX path (supervisor.py:1256-1278) adds `Popen` handles to an `_ORPHANS` set and later calls bare `p.kill()` at process-exit (`atexit`) — this is a secondary, best-effort cleanup path (not the primary halt path), separate from `_wd_kill_tree`. Decide in planning whether `_reap_orphans()` also needs `os.killpg()` (same fix, same shape) or whether the primary `_wd_kill_tree()` fix is sufficient since `_leash` only fires on already-closed processes at backend shutdown. Given D-06's scope statement covers `supervisor.py`'s "4 spawn/kill sites," the intended reading is `_wd_kill_tree` alone (the fix propagates to all 4 spawn sites through it) plus the `start_new_session` additions at all 4 `Popen` calls — `_reap_orphans` is likely out of the committed 4-site scope but worth a one-line note in the plan if the executor finds it easy to include.

---

## Shared Patterns

### POSIX process-group spawn + immediate SIGKILL termination
**Source:** `engine/backend/orgtree/gitrunner.py:36-48,67-72`
**Apply to:** `codexrun.py`, `antigravityrun.py`, `supervisor.py` (4 spawn sites + `_wd_kill_tree`), and conditionally `mailhub_runtime.py`
```python
# spawn:
subprocess.Popen(argv, ..., start_new_session=(os.name != "nt"))
# kill:
if os.name == "nt":
    subprocess.run(["taskkill", "/PID", str(proc.pid), "/T", "/F"], ...)
else:
    try:
        os.killpg(proc.pid, signal.SIGKILL)
    except ProcessLookupError:
        pass
if proc.poll() is None:
    proc.kill()
proc.wait(timeout=5)
```
No grace period (`SIGTERM`-then-wait) anywhere in this pattern — immediate `SIGKILL` is the deliberate, existing, correct behavior (D-05/D-06). Do not introduce a grace period.

### Resolver fallback chain (verification-only reference, no new code expected)
**Source:** `engine/backend/orgtree/providers.py:447-458` (`codex_path()`)
```python
def codex_path() -> tuple[str | None, str]:
    env = os.environ.get("ORGTREE_CODEX")
    if env:
        return env, "env"
    pin = _codex_pin()
    if pin:
        return pin, "pin"
    onpath = shutil.which("codex")
    if onpath:
        return onpath, "path"
    return None, ""
```
Used only as the shape reference confirming Claude's `_claude_argv()`/`CLAUDE` in `supervisor.py` already follows an equivalent (differently-structured but functionally complete) resolution chain — no new `claude_path()` needed per D-02's resolution.

## No Analog Found

None — every file in scope already has `gitrunner.py`'s POSIX pattern as a direct, provably-correct in-repo analog. This phase is a pure propagate-the-existing-fix task, not new pattern design.

## Metadata

**Analog search scope:** `engine/backend/orgtree/` (gitrunner.py, providers.py, codexrun.py, antigravityrun.py, supervisor.py, liveness.py), `engine/mailhub_runtime.py`
**Files scanned:** 7 read directly, line-targeted (all >400 lines except gitrunner.py; supervisor.py is 32,746 lines — read via targeted non-overlapping line ranges only, no full-file load)
**Pattern extraction date:** 2026-09-17
