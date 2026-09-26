---
phase: 05-end-to-end-verification-test-coverage
plan: 01
subsystem: testing
tags: [electron, acceptance-testing, macos, claude-code-cli, process-lifecycle]
requires: []
provides: [real-agent-turn-acceptance-runner, macos-preflight-gate, macos-hardware-verification-checklist]
affects: [tests/acceptance, tools, docs/qa]
tech-stack:
  added: []
  patterns: [DOM-driven Electron acceptance harness (application.cjs shape), authorization-gated real-provider test lane (run_claude.py shape), static source pre-flight gate]
key-files:
  created:
    - tests/acceptance/run_agent_turn.mjs
    - tests/acceptance/agent_turn.cjs
    - tools/verify-macos-preflight.mjs
    - docs/qa/macos-hardware-verification.md
  modified:
    - package.json
decisions:
  - "chartObserved evidence is read from the real chat-transcript API (/api/orgs/{org}/nodes/{node}/chat), not an engine-internal ASGI monkeypatch like run_claude.py's tool-observations.jsonl - the real Electron app always spawns the unmodified engine/launch.py via a hardcoded path in apps/desktop/main/engine.ts, so that monkeypatch technique cannot reach the real spawn path without touching production engine code (out of this plan's scope)."
  - "All three process-lifecycle blockers named in this plan's threat model (T-05-02, T-05-03) are already fixed in current source - the preflight checker was written to test the correct invariants rather than assert a stale premise, and it reports PASS today instead of the FAIL the plan predicted."
metrics:
  duration: ~1h
  completed: 2026-09-23
status: complete
actuals:
  tokens: 12000
  tasks: 3
  commits: 3
---

# Phase 5 Plan 01: End-to-End Real-Agent-Turn Acceptance Runner Summary

Ported `run_claude.py`'s proven hire/message/observe pass criteria into an Electron DOM-driven acceptance harness matching `application.cjs`'s shape, added a static macOS process-lifecycle preflight gate, and recorded the one verification step no CI run can prove - with source inspection during Task 2 finding all three process-lifecycle blockers this plan's threat model named are already fixed in current source.

## What Was Built

1. **`tests/acceptance/run_agent_turn.mjs`** - mirrors `run.mjs`'s orchestration shape (isolated root, prerequisite checks, harness preflight, runtime manifest digest, platform-correct Electron/Python resolution). Gated behind `ORGTREE_ACCEPTANCE_AUTHORIZE_AGENT_TURN=1`; prints `{"status":"INERT",...}` and exits 2 without it, so it is safe to leave wired into `npm run` today. Wires `tools/verify-macos-preflight.mjs` as a required first step of the real-provider phase.

2. **`tests/acceptance/agent_turn.cjs`** - the Electron-side driver, following `application.cjs`'s `evaluate()`/`waitFor()`/`check()` harness shape. Loads the real, unmodified `dist/main/index.cjs` entrypoint, creates an organization through the real onboarding form (`organization_flow.cjs`), hires one `tier: "haiku"` agent with a bounded tool-restricted charter, sends it a deterministic task through the real `.cc-composer` chat input, and polls the org's own `/chat` API for the `V2_ACCEPTANCE_RESULT` text and exactly one `orgtree_chart` tool call.

3. **`tools/verify-macos-preflight.mjs`** - three static source checks (no live process spawning) against `codexrun.py`, `antigravityrun.py`, and `process_lifetime.py`. Exits 0/PASS when the required invariants (`start_new_session`, POSIX process-group kill via `os.killpg`, no unconditional non-Windows raise) hold, 1/FAIL naming each unmet one otherwise.

4. **`docs/qa/macos-hardware-verification.md`** - six-step manual checklist for the one thing no CI run can prove: a real Apple Silicon Mac completing one real agent job with no manual workaround beyond the already-documented PKG-03 Gatekeeper right-click-to-Open step. Result section is templated and unfilled, correctly blocked until Phase 1-4 ship a real hardware target.

5. **`package.json`** - added `test:acceptance:agent-turn` and `test:macos-preflight` scripts.

## Deviations from Plan

### Auto-fixed Issues

None - no bugs found in code this plan touched.

### Significant Finding (documented, not silently worked around)

**1. All three process-lifecycle blockers this plan's threat model named are already fixed in current source.**

During Task 2, direct source inspection found:

- `engine/backend/orgtree/codexrun.py:513` already passes `start_new_session=(os.name != "nt")` to its `app-server` `subprocess.Popen(...)` call, and `close()` (line 958) already kills the POSIX process group via `os.killpg(self.proc.pid, signal.SIGKILL)` before the belt-and-suspenders `self.proc.kill()`.
- `engine/backend/orgtree/antigravityrun.py:762` already passes the same `start_new_session` kwarg, and `kill_tree()` (line 580) already calls `os.killpg(proc.pid, signal.SIGKILL)` on POSIX.
- `engine/process_lifetime.py`'s `arm_process_lifetime` has full POSIX branches throughout (env allowlist at line 43-44, `start_new_session=os.name != "nt"` at line 52, and the guardian's own POSIX signal-based watch loop at line 382) - no unconditional `RuntimeError` for `os.name != "nt"` exists anywhere in the file. A repo-wide search for the plan's cited error text (`"process ownership is currently supported on Windows only"`) found zero matches anywhere in the codebase.

This directly contradicts the plan's `<objective>` claim (based on RESEARCH.md and a "newly-discovered gap" note) that these three gaps currently block macOS boot, and contradicts Task 2's `<done>` criterion expecting the checker to report FAIL today. `tools/verify-macos-preflight.mjs` was written to test the *correct* invariants (matching the property each check is actually meant to guard) rather than assert a now-stale premise, so it correctly reports `{"status":"PASS"}` / exit 0 against current source - a better outcome for the project, but one that means Task 2's own automated `<verify>` (`test $? -eq 1 && echo CORRECTLY_DETECTS_KNOWN_GAPS_TODAY`) does not produce that echo today. Ran and confirmed: `node tools/verify-macos-preflight.mjs` exits 0 with `{"status":"PASS"}`.

Likely explanation: the plan's `.planning/STATE.md` snapshot at planning time read `completed_phases: 0`, but this worktree's actual `.planning/STATE.md` shows `completed_phases: 1` (Phase 3 Plan 2 of 2 in progress) and the git history shows Phase 4 (macOS UI/OS integration parity) already merged - the process-lifecycle fixes evidently landed as part of that already-merged work, after RESEARCH.md was written.

Flagging for the user/next phase: since PROC-03/Phase 2 was the stated owner of checks 1-2, worth confirming whether Phase 2's plan already accounts for this being done, so it isn't re-planned as outstanding work.

**2. `chartObserved` evidence source could not literally match `run_claude.py`'s `tool-observations.jsonl` mechanism.**

`run_claude.py`/`claude_engine.py` produce that file via an ASGI-level monkeypatch of `engine.launch.load_app`, invoked by running `claude_engine.py` directly as a standalone Python entrypoint (bypassing Electron/`engine.ts` entirely). The real Electron app's process spawn (`apps/desktop/main/engine.ts:204`) hardcodes `path.join(options.directory, 'launch.py')` with no environment-variable override for the script filename, so there is no way to substitute an instrumented entrypoint through the real app's spawn path without modifying `engine.ts` or `engine/launch.py` - both out of this plan's file scope and a larger change than this task warrants. `agent_turn.cjs` instead observes the same evidence (arithmetic result text and exactly one `orgtree_chart` tool call) through the org's own `/api/orgs/{org}/nodes/{node}/chat` transcript API, called via the renderer's real `fetch` (same-origin, same mechanism `organization_flow.cjs` already uses) - reusing an existing, non-invasive, already-real API surface rather than adding engine instrumentation. This is inferred from codebase evidence (`turnread.py`'s `assistant_shape()` extracts `tool_use` block names into a `tools` list; `chat_window.py`'s `_project_tail` reads raw `message.content` blocks including `tool_use` type/id) rather than verified against real engine output, since no macOS package exists yet to run this against. May need adjustment once Phase 1-2 unblock a real end-to-end run.

### Blocked (as planned)

- Task 1 and Task 3 cannot reach a real `PASS`/filled-in Result until Phase 1 ships a macOS package target (PKG-01) and later phases land provider fixes - exactly as the plan anticipated. Verified today: `node tests/acceptance/run_agent_turn.mjs` reports `{"status":"INERT","reason":"Explicit agent-turn authorization required"}` and exits 2.

## Known Stubs

None beyond the plan's own explicitly-templated, intentionally-unfilled `docs/qa/macos-hardware-verification.md` Result section (documented above as blocked, not a stub needing resolution in this phase).

## Verification

- `node tools/verify-macos-preflight.mjs` (also `npm run test:macos-preflight`) - exit 0, `{"status":"PASS"}`.
- `node tests/acceptance/run_agent_turn.mjs` (also `npm run test:acceptance:agent-turn`) - exit 2, `{"status":"INERT","reason":"Explicit agent-turn authorization required"}`.
- `test -f docs/qa/macos-hardware-verification.md && grep -c "no manual workarounds"` - file exists, count 2.
- `node --check` passed on both new `.mjs`/`.cjs` files.

## Threat Flags

None - this plan's own `<threat_model>` already covered the new surface (T-05-01 through T-05-04); no additional untracked surface was introduced.

## Self-Check: PASSED

- FOUND: tests/acceptance/run_agent_turn.mjs
- FOUND: tests/acceptance/agent_turn.cjs
- FOUND: tools/verify-macos-preflight.mjs
- FOUND: docs/qa/macos-hardware-verification.md
- FOUND: 434a086 (feat: real-agent-turn acceptance runner)
- FOUND: 3bad8ac (feat: macOS process-lifecycle preflight gate)
- FOUND: 13f4acd (docs: macOS hardware verification checklist)
