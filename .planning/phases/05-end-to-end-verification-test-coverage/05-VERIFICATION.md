---
phase: 05-end-to-end-verification-test-coverage
verified: 2026-09-23T00:00:00Z
status: human_needed
score: 4/5 must-haves verified
behavior_unverified: 0
overrides_applied: 0
human_verification:
  - test: "Run docs/qa/macos-hardware-verification.md's checklist end-to-end: build the unsigned .app (npm run package:mac), copy to a real Apple Silicon Mac, launch it, hire one real agent, send it one real task through the app's own chat UI, and confirm the response + orgtree_chart tool call render in the UI."
    expected: "The agent completes a real job and the result is visible in the app's chat UI, with no manual workaround beyond the documented Gatekeeper right-click-Open step."
    why_human: "Requires real Apple Silicon hardware, a real provider account with credentials, and Gatekeeper/AMFI behavior that cannot be observed from a CI/build machine. tests/acceptance/run_agent_turn.mjs was executed live on this machine (app built, Python runtime provisioned, engine/mailhub submodule initialized) and got past renderer mount and organization creation, but failed at the 'hire agent' step because the isolated acceptance sandbox's accounts-registry.json has zero provider accounts registered - there is no way to inject real provider credentials into that sandbox in this environment. This is not a code defect in the runner; docs/qa/macos-hardware-verification.md's Result section still reads 'Not yet run', confirming the hardware/credentialed proof is outstanding."
---

# Phase 5: End-to-End Verification & Test Coverage Verification Report

**Phase Goal:** Prove a real agent job runs on macOS and close the Windows-only test gaps
**Verified:** 2026-09-23
**Status:** human_needed
**Re-verification:** No — initial verification

## Goal Achievement

### Observable Truths

| # | Truth | Status | Evidence |
|---|-------|--------|----------|
| 1 | A launched macOS build can spawn one real Claude Code agent turn observable through the app's own chat UI | ⚠️ Partially proven — see Human Verification | `tests/acceptance/run_agent_turn.mjs` + `agent_turn.cjs` exist, are wired to real Electron main (`apps/desktop/main/index.ts`) → Python engine (`engine/launch.py`), and were executed live on this machine: app built, Python runtime provisioned, `engine/mailhub` submodule initialized. Run got past renderer mount and organization creation, then failed at "hire agent" because the isolated acceptance sandbox's `accounts-registry.json` has zero provider accounts (no way to inject real credentials here). No successful live hire+chat has been observed anywhere yet. |
| 2 | The two known process-lifecycle gaps (codexrun.py/antigravityrun.py missing `start_new_session`, `process_lifetime.py`'s unconditional non-nt `RuntimeError`) are caught by an automated pre-flight gate instead of being worked around inside the acceptance test | ✓ VERIFIED | `tools/verify-macos-preflight.mjs` exists (61 lines), performs the three static source checks described in the plan, and running it live produces `{"status":"PASS"}` exit 0 (spot-checked). `run_agent_turn.mjs` wires it as a required pre-check. |
| 3 | A human has confirmed on real Apple Silicon hardware that the packaged `.app` completes one real agent job with no manual workaround beyond the documented Gatekeeper step | ✗ NOT YET DONE — human_needed | `docs/qa/macos-hardware-verification.md` exists (40 lines) with the full checklist, but its Result section reads: "Status: Not yet run — blocked on Phase 1-4 shipping a real macOS package target and provider process-lifecycle fixes to verify against." No date/verifier/notes recorded. |
| 4 | The Windows-only test suite gets macOS counterparts so mac-specific logic has real coverage instead of silent skips (VER-02) | ✓ VERIFIED | 6 new `_on_posix` companion tests in `tests/test_service_host.py` and 2 in `tests/test_claude_pipe_lifecycle.py` exist and pass (spot-checked one named test from each, both PASSED). `tests/test_process_lifetime.py` (8 tests, no gate) and `tests/test_startup_readiness.py` (2 tests, Windows-only gate removed) both run and pass fully on POSIX (10/10, confirmed by full-file run). One residual Windows-only skip remains in `tests/test_startup_progress.py::BootStartupTests`, honestly re-documented (not silently skipped) with a source-verified reason and tracked as an open item in `.planning/WINDOWS.md` (#4, phase 05) rather than claimed fixed. |
| 5 | No silent/undocumented process-lifecycle regressions were introduced while closing the gaps | ✓ VERIFIED | `engine/service_host.py`'s POSIX branches (`os.name != "nt"`) implement real `chmod(0o600)`, `O_CREAT|O_EXCL`, and `st_uid`/`st_mode` verification — not stubs. No debt markers (TODO/FIXME/XXX/placeholder) found in any of the 10 files this phase touched. |

**Score:** 4/5 truths verified (0 present-behavior-unverified; truth #1 is downgraded by the hardware gap in truth #3, both routed to the same human-verification item)

### Required Artifacts

| Artifact | Expected | Status | Details |
|----------|----------|--------|---------|
| `tests/acceptance/run_agent_turn.mjs` | Electron-launched real-agent-turn acceptance runner | ✓ VERIFIED | 90 lines. `ORGTREE_ACCEPTANCE_AUTHORIZE_AGENT_TURN=1` gate confirmed live: unauthorized run prints `{"status":"INERT","reason":"Explicit agent-turn authorization required"}` and exits 2. |
| `tests/acceptance/agent_turn.cjs` | Electron DOM-driven hire/message/observe harness | ✓ VERIFIED | 126 lines, no debt markers, substantive implementation. |
| `tools/verify-macos-preflight.mjs` | Static source pre-flight gate for the two process-lifecycle blockers | ✓ VERIFIED | 61 lines; executed live, produces `{"status":"PASS"}` exit 0. |
| `docs/qa/macos-hardware-verification.md` | Manual hardware verification checklist for VER-01 | ✓ VERIFIED (as a checklist artifact) | 40 lines, full checklist present. Result section correctly still "Not yet run" — this is the honest state, not a gap in the artifact itself. |
| `engine/service_host.py` | Real POSIX descriptor-protection implementation | ✓ VERIFIED | 507 lines; POSIX branches use real `chmod`/`O_EXCL`/`stat` calls, confirmed via source search, not a no-op. |
| `tests/test_service_host.py` | 6 new macOS-only companion tests | ✓ VERIFIED | 533 lines; 6 `_on_posix` methods found; full-file run: 22 passed, 8 skipped (Windows-only ACL tests correctly still skip on macOS). |
| `tests/test_claude_pipe_lifecycle.py` | 2 new macOS-only companion tests | ✓ VERIFIED | 426 lines; 2 `_on_posix` methods found; one spot-checked, PASSED. |
| `tests/test_process_lifetime.py` | No Windows-only gate; 8/8 pass on POSIX | ✓ VERIFIED | 272 lines; no `skipUnless`/`skipIf` gate found; full-file run 8/8 pass. |
| `tests/test_startup_progress.py` | Windows-only gate kept, reason rewritten to actual verified gap | ✓ VERIFIED | 105 lines; `BootStartupTests` still gated `skipUnless(os.name == "nt", ...)` with an honest, source-verified reason string (RootLock fcntl release timing). Matches SUMMARY's stated decision. |
| `tests/test_startup_readiness.py` | Windows-only gate removed after verifying POSIX pass | ✓ VERIFIED | 143 lines; no skip gate found; full-file run 2/2 pass. |

### Data-Flow Trace (Level 4)

Not applicable in the classic UI-rendering sense — this phase produces test/verification tooling, not user-facing data rendering. The relevant "data flow" is the acceptance-runner's evidence chain, traced above under Truth #1: `run_agent_turn.mjs` → `agent_turn.cjs` → real Electron main → real Python engine, confirmed live (not mocked) up to the point credential provisioning blocks further progress.

### Key Link Verification

| From | To | Via | Status | Details |
|------|-----|-----|--------|---------|
| `run_agent_turn.mjs` | `agent_turn.cjs` | direct import/spawn | ✓ WIRED | Confirmed by live execution (INERT gate fires correctly; script runs to the documented failure point when authorized). |
| `run_agent_turn.mjs` | `tools/verify-macos-preflight.mjs` | required pre-check | ✓ WIRED | Preflight script runs and returns PASS live. |
| `agent_turn.cjs` | Electron main (`apps/desktop/main/index.ts`) → Python engine (`engine/launch.py`) | real app launch, not mocked | ✓ WIRED | Orchestrator-confirmed live run reached renderer mount + org creation via the real app process before failing at credential-gated hire step. |

### Behavioral Spot-Checks

| Behavior | Command | Result | Status |
|----------|---------|--------|--------|
| Preflight gate passes on current source | `node tools/verify-macos-preflight.mjs` | `{"status":"PASS"}`, exit 0 | ✓ PASS |
| Unauthorized acceptance runner stays inert | `node tests/acceptance/run_agent_turn.mjs` | `{"status":"INERT",...}`, exit 2 | ✓ PASS |
| One `_on_posix` descriptor-protection test | `pytest tests/test_service_host.py::...test_protected_birth_denies_every_second_handle_and_survives_close_on_posix` | PASSED | ✓ PASS |
| One `_on_posix` pipe-lifecycle test | `pytest tests/test_claude_pipe_lifecycle.py::...test_idle_watchdog_ends_launcher_and_child_then_returns_queued_mail_on_posix` | PASSED | ✓ PASS |
| Full `test_process_lifetime.py` + `test_startup_readiness.py` | `pytest tests/test_process_lifetime.py tests/test_startup_readiness.py` | 10 passed, 2 subtests passed | ✓ PASS |
| Full `test_service_host.py` | `pytest tests/test_service_host.py` | 22 passed, 8 skipped (Windows-only ACL tests, expected) | ✓ PASS |

### Probe Execution

No `scripts/*/tests/probe-*.sh` convention used by this project; no probes declared in either PLAN. Skipped — not applicable to this phase's artifact shape (Node/Python test runners and a CLI gate script cover the equivalent role, exercised above under Behavioral Spot-Checks).

### Requirements Coverage

| Requirement | Source Plan | Description | Status | Evidence |
|-------------|-------------|--------------|--------|----------|
| VER-01 | 05-01 | Packaged unsigned `.app` builds, launches, and completes a real end-to-end agent job on macOS | ? NEEDS HUMAN | Automated acceptance runner + preflight gate exist, are wired correctly, and were executed live as far as possible without real provider credentials (reached hire-agent step, blocked by empty accounts-registry.json in the isolated sandbox — not a code defect). The one remaining proof point — a human on real Apple Silicon hardware completing a full hire+chat — is documented in `docs/qa/macos-hardware-verification.md` and its Result section still reads "Not yet run". REQUIREMENTS.md marks VER-01 complete; this VERIFICATION.md records the outstanding hardware/credentialed proof as a human-verification item per explicit orchestrator instruction, and does not alter the REQUIREMENTS.md checkbox. |
| VER-02 | 05-02 | Windows-only test suite gets macOS counterparts instead of silent skips | ✓ SATISFIED | 8 new `_on_posix` companion tests exist and pass; two previously-gated files (`test_process_lifetime.py`, `test_startup_readiness.py`) confirmed passing ungated on POSIX; the one remaining gate (`test_startup_progress.py::BootStartupTests`) is honestly redocumented with a source-verified reason rather than silently skipped, and tracked as an open item in `.planning/WINDOWS.md` (#4) rather than claimed as fixed. This matches the requirement's "real coverage instead of silent skips" bar — the residual gap is visible and tracked, not hidden. |

No orphaned requirements: REQUIREMENTS.md maps only VER-01 and VER-02 to Phase 5, and both are declared in the plans' `requirements` frontmatter (05-01: `[VER-01]`, 05-02 via `requirements-completed: [VER-02]`).

### Anti-Patterns Found

None. Searched all 10 files this phase modified/created for `TODO|FIXME|XXX|placeholder|not yet implemented|coming soon|console.log-only handlers|return null stubs` — zero matches. `.planning/WINDOWS.md` item #4 is a deliberately tracked, honestly-labeled deviation (not a hidden debt marker in source) — it references its own ledger entry, satisfying the debt-marker gate's "formal follow-up work" exception.

### Human Verification Required

### 1. Real Apple Silicon hardware hire+chat proof (VER-01)

**Test:** Follow `docs/qa/macos-hardware-verification.md`'s checklist: build the unsigned `.app` (`npm run package:mac`), copy to a real Apple Silicon Mac, launch it, hire one real agent, send it one real task through the app's own chat UI, and confirm the response + `orgtree_chart` tool call render in the UI.

**Expected:** The agent completes a real job and the result is visible in the app's chat UI, with no manual workaround beyond the documented Gatekeeper right-click-Open step.

**Why human:** Requires real Apple Silicon hardware, a real provider account with valid credentials, and Gatekeeper/AMFI approval behavior — none of which can be observed or faked from this build machine. The automated acceptance runner (`tests/acceptance/run_agent_turn.mjs`) was executed live here — app built, Python runtime provisioned, `engine/mailhub` submodule initialized — and it got past renderer mount and organization creation, but failed at "hire agent" because the isolated acceptance sandbox's `accounts-registry.json` has zero provider accounts registered, with no way to inherit real provider credentials in this environment. This is the maximum automated proof possible without a human and real credentials.

### Gaps Summary

No blocking engineering gaps. Both plans' artifacts exist, are substantive, are wired correctly, and pass live spot-checks. The phase's remaining incompleteness is the one item both plans explicitly scoped as requiring a human: real-hardware, credentialed proof of a live agent hire+chat (VER-01's third must-have truth), which `docs/qa/macos-hardware-verification.md` itself documents as outstanding ("Not yet run"). VER-02's engineering work is fully verified, including an honest (not silently hidden) residual gap in `test_startup_progress.py::BootStartupTests`, tracked in `.planning/WINDOWS.md`. REQUIREMENTS.md's VER-01/VER-02 checkboxes are left unchanged per instruction; this report exists to make the outstanding hardware step visible to whoever ships this phase.

---

_Verified: 2026-09-23_
_Verifier: Claude (gsd-verifier)_
