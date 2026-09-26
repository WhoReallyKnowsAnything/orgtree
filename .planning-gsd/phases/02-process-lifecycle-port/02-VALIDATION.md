---
phase: "2"
slug: "process-lifecycle-port"
# status lifecycle: draft (seeded by plan-phase) → validated (set by validate-phase §6)
# audit-milestone §5.5 distinguishes NOT-VALIDATED (draft) from PARTIAL (validated + nyquist_compliant: false) (#2117)
status: draft
nyquist_compliant: false
wave_0_complete: false
created: "2026-09-17"
---

# Phase 2 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Python `unittest` (stdlib) — no pytest.ini/pyproject.toml/conftest.py found |
| **Config file** | none — Wave 0 |
| **Quick run command** | `python -m unittest tests.test_provider_attempt_and_liveness -v` |
| **Full suite command** | `python -m unittest discover -s tests -p 'test_*.py'` |
| **Estimated runtime** | ~30 seconds (full suite; no benchmarked figure in RESEARCH.md) |

---

## Sampling Rate

- **After every task commit:** Run `python -m unittest tests.test_provider_attempt_and_liveness -v`
- **After every plan wave:** Run `python -m unittest discover -s tests -p 'test_*.py'`
- **Before `/gsd-verify-work`:** Full suite must be green, plus the new real-process-tree integration test(s) from Wave 0
- **Max feedback latency:** 30 seconds

---

## Per-Task Verification Map

Task IDs are assigned by the planner — this table is filled in during planning, with each task's `<automated>` verify command mapped to one of the requirement-level tests below.

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| TBD | TBD | TBD | PROC-01 | — | `codex_path()`/`antigravity_path()`/`supervisor.CLAUDE`/`_claude_argv()` resolve correctly on macOS | unit (mocked path) | `python -m unittest tests.test_codex_home_usage -v` | ✅ codex only — antigravity/claude need Wave 0 | ⬜ pending |
| TBD | TBD | TBD | PROC-02 | — | `os.kill(pid, 0)` distinguishes alive/dead/reaped correctly | unit (real spawn+reap) | `python -m unittest tests.test_provider_attempt_and_liveness -v` | ✅ `test_3e_a_live_process_is_alive_and_a_reaped_one_is_dead` already covers this | ⬜ pending |
| TBD | TBD | TBD | PROC-03 | — | Killing a provider CLI (codexrun.py, antigravityrun.py, supervisor.py) kills its descendants on POSIX, no orphans | integration (real process tree) | none exists today | ❌ Wave 0 — every existing halt/kill test mocks `_wd_kill_tree` entirely | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] Real-process-tree termination test for `codexrun.py::CodexProcess.close()` — spawn a shell that forks a child, call `close()`, assert the grandchild is also gone. No such test exists today; every current test mocks the kill call entirely.
- [ ] Same real-process-tree test for `antigravityrun.py::kill_tree()`.
- [ ] Same real-process-tree test for `supervisor.py`'s Claude-CLI kill path (per D-07's scope expansion), covering all 4 spawn/kill sites via the shared `_wd_kill_tree()`/`_reap_orphans()` helpers.
- [ ] Mocked-path resolution test for `antigravity_path()` and for `supervisor.CLAUDE`/`_claude_argv()`, mirroring `test_codex_home_usage.py`'s existing pattern for `codex_path()` — neither currently exists.

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| `antigravity_path()`'s `~/.local/bin/agy` PATH-fallback branch | PROC-01 | Antigravity CLI is not installed on the dev machine used for research; mocked-path unit test covers the logic, but real-binary confirmation needs a machine with Antigravity installed | Install Antigravity on a real macOS machine, run the app, confirm the agent picker resolves and launches the Antigravity provider without a manual path override |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 30s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
