---
phase: 01-packaging-runtime-foundation
plan: 03
subsystem: packaging
tags: [electron-builder, macos, codesign, gatekeeper, pythondontwritebytecode]

requires:
  - phase: 01-packaging-runtime-foundation (plan 01)
    provides: engine/runtime provisioned python3.13 (RUN-01)
  - phase: 01-packaging-runtime-foundation (plan 02)
    provides: apps/desktop/assets/orgtree-eye.icns

provides:
  - "package.json build.mac: ad-hoc-signed arm64 dmg/zip targets, afterPack hook, package:mac(:dir|:x64) scripts"
  - tools/sign-runtime-macos.mjs, an afterPack hook that ad-hoc-signs the bundled runtime tree electron-builder's own signer never reaches
  - PYTHONDONTWRITEBYTECODE=1 on both the packaged engine spawn (apps/desktop/main/engine.ts) and its guardian subprocess (engine/process_lifetime.py), fixing the PKG-03 seal-invalidation regression found post-pause
  - docs/macos-first-launch.md documenting the in-place vs quarantined-copy Gatekeeper flows, with --verify --deep --strict as the correct check

affects: []

actuals:
  tokens: unmeasured (resumed session, ran inline after a worktree-isolation degrade — see Deviations)
  tasks: 4 (3 completed pre-pause, 1 blocked on human hardware verification)
  commits: 9

tech-stack:
  added: []
  patterns: ["PYTHONDONTWRITEBYTECODE=1 on every spawn/re-exec of the packaged interpreter, set explicitly rather than relied on to inherit, since a security-motivated env allowlist for a child process silently drops unlisted vars"]

key-files:
  created:
    - docs/macos-first-launch.md
    - tools/sign-runtime-macos.mjs
  modified:
    - package.json
    - apps/desktop/main/engine.ts
    - engine/process_lifetime.py
    - .planning/phases/01-packaging-runtime-foundation/01-03-PLAN.md

key-decisions:
  - "Fixed PKG-03's seal invalidation with PYTHONDONTWRITEBYTECODE=1 rather than a build-time .pyc-precompile-and-reseal step: the write happens at launch (import time), not build time, so suppressing the write is simpler than re-signing after every launch — and re-signing after launch was explicitly the pattern to avoid (ordering: nothing may launch the app between signing and the final verify)."
  - "Root cause had two independent spawn sites, not one: apps/desktop/main/engine.ts's direct spawn of the packaged interpreter, and engine/process_lifetime.py's guardian watchdog, which re-execs sys.executable with a deliberately strict env allowlist (PATH/HOME/TMPDIR/LANG/LC_ALL only, to avoid leaking desktop/provider credentials into it) that silently dropped PYTHONDONTWRITEBYTECODE even after the first fix. Both needed the env var set explicitly; fixing only the first left 8 stdlib __pycache__ dirs written by the guardian on every launch, still breaking strict verify."
  - "Replaced Task 2's codesign -dv --verbose=4 verify step (and the matching check in docs/macos-first-launch.md) with codesign --verify --deep --strict: -dv only proves a signature exists, not that it's still valid, which is exactly how this bundle passed pre-launch and silently broke post-launch across the original pre-pause execution."

patterns-established:
  - "Any code path that re-invokes the packaged python interpreter (direct spawn, watchdog re-exec, future subprocess) must set PYTHONDONTWRITEBYTECODE=1 explicitly in its own env construction — do not assume env inheritance covers it once anything builds a filtered env dict."

requirements-completed: [PKG-01, PKG-03]

coverage:
  - id: D1
    description: "package.json build.mac exists (identity '-', correct icon, arm64-only targets, afterPack hook); package:mac:dir/package:mac/package:mac:x64 scripts exist; package:win/package:dir unchanged"
    requirement: "PKG-01"
    verification:
      - kind: unit
        ref: "node -e assertion on package.json build.mac (Task 1 verify)"
        status: pass
      - kind: other
        ref: "npm run package:mac:dir succeeds end-to-end, produces release/mac-arm64/Orgtree.app"
        status: pass
    human_judgment: false
  - id: D2
    description: "Every nested runtime binary is ad-hoc-signed by tools/sign-runtime-macos.mjs's afterPack hook, in addition to electron-builder's own top-level signing pass"
    requirement: "PKG-01"
    verification:
      - kind: other
        ref: "codesign --verify --deep --strict release/mac-arm64/Orgtree.app — pass, fresh build"
        status: pass
    human_judgment: false
  - id: D3
    description: "codesign --verify --deep --strict passes on the built .app both BEFORE and AFTER launching it (the actual PKG-03 seal-invalidation bug)"
    requirement: "PKG-03"
    verification:
      - kind: other
        ref: "Reproduced the full launch.py + guardian child tree directly (same argv/env as apps/desktop/main/engine.ts's spawn) against a fresh --dir build: 0 __pycache__ dirs and strict verify pass both before and after a full startup/shutdown cycle (ready event observed, guardian armed and torn down cleanly)"
        status: pass
    human_judgment: false
  - id: D4
    description: "A quarantined copy of the .app shows the recoverable 'developer cannot be verified' Gatekeeper dialog on real Apple Silicon hardware, not 'is damaged'; RUN-01's engine child tree still comes up with no regression; Intel verification if hardware is available"
    requirement: "PKG-03"
    verification:
      - kind: other
        ref: "spctl -a --type execute --raw against an xattr-quarantined copy: assessment:verdict=false (Gatekeeper rejects it, consistent with either dialog wording)"
        status: pass
      - kind: human
        ref: "Task 4 (checkpoint:human-verify, gate=blocking-human), run by the user on real Apple Silicon: in-place launch showed no Gatekeeper/AMFI dialog (one macOS TCC file-access privacy prompt appeared instead — see Deviations); codesign --verify --deep --strict still passed after quitting; a ditto+xattr-quarantined copy showed the recoverable 'cannot be verified' dialog, not 'is damaged'"
        status: pass
    human_judgment: true
    rationale: "The specific Gatekeeper dialog text is a Finder/LaunchServices GUI behavior not exposed via any CLI assessment tool; spctl confirmed Gatekeeper blocks the quarantined copy (consistent with the fix working), and the user's real-hardware run confirmed the actual dialog wording. Intel hardware was not available; x64 verification remains a documented follow-up per the plan's own allowance."

duration: resumed session (paused 2026-09-17, resumed and closed 2026-09-23)
completed: 2026-09-23
status: complete
---

# Phase 01-03: Package and Sign the macOS App (PKG-01/PKG-03) Summary

**Fixed the PKG-03 seal-invalidation blocker that paused this phase: two independent code paths spawned/re-exec'd the packaged Python interpreter without suppressing bytecode writes, breaking `codesign --verify --deep --strict` after every launch. Both are fixed and verified via a full engine + guardian launch reproduction, and confirmed by the user on real Apple Silicon (Task 4: PASS). Phase 1 is complete.**

## Performance
- **Tasks:** 3 of 4 completed pre-pause (Tasks 1–3, committed 2026-09-17); this session added 2 fix commits + 1 docs commit and closed Task 4 (blocking-human checkpoint) via the user's real-hardware run.
- **Files modified this session:** 4 (`apps/desktop/main/engine.ts`, `engine/process_lifetime.py`, `.planning/phases/01-packaging-runtime-foundation/01-03-PLAN.md`, `docs/macos-first-launch.md`)

## Accomplishments
- Root-caused PKG-03: the packaged `.app`'s `Contents/Resources/engine/` writes `__pycache__/*.pyc` on import because it's a live, writable location, and CPython writes bytecode next to `.py` files by default — invalidating the code signature seal on first launch.
- Found and fixed **both** spawn sites that write bytecode: the main engine spawn in `apps/desktop/main/engine.ts` (`PYTHONDONTWRITEBYTECODE=1` added to its env), and the guardian watchdog re-exec in `engine/process_lifetime.py::arm_process_lifetime` (same var forced into its deliberately-restricted allowlisted env). Fixing only the first left the guardian writing 8 stdlib `__pycache__` dirs on every launch.
- Verified against a real `npm run package:mac:dir` build: `codesign --verify --deep --strict` passes before build-completion and after a full launch/shutdown cycle of the real engine + guardian child tree (ready event observed, clean teardown, 0 `__pycache__` anywhere in the bundle).
- Replaced the plan's and doc's `codesign -dv` verify checks with `--verify --deep --strict` — `-dv` only proves a signature exists, which is exactly how the original pre-pause execution passed Task 2's automated check while shipping a bug that broke the seal on first real launch.
- Confirmed no regression: PKG-02's icon (`icon.icns`) is present in the built bundle; RUN-01's full engine/guardian process tree starts and tears down cleanly.

## Task Commits
1. Task 1: electron-builder mac target and package scripts (PKG-01) — `e99b9c3` (feat, pre-pause)
2. Task 2: afterPack ad-hoc signing hook (PKG-01) — `9113738` (feat, pre-pause)
3. Task 3: document first-launch Gatekeeper/AMFI flow (PKG-03) — `c22ab20` (docs, pre-pause)
4. (interim) resolve packaged macOS python path (PKG-01) — `ba4dea5` (fix, pre-pause)
5. PKG-03 fix, engine spawn — `6175497` (fix, this session)
6. PKG-03 fix, guardian subprocess env — `d9021bd` (fix, this session)
7. Replace `-dv` verify checks with `--verify --deep --strict` — `fd5f9fb` (docs, this session)

## Files Created/Modified
- `package.json` — `build.mac` config, `package:mac(:dir|:x64)` scripts (Task 1, pre-pause)
- `tools/sign-runtime-macos.mjs` — afterPack signing hook (Task 2, pre-pause)
- `docs/macos-first-launch.md` — created (Task 3, pre-pause); updated this session to use `--verify --deep --strict` and note the `PYTHONDONTWRITEBYTECODE` fix
- `apps/desktop/main/engine.ts` — added `PYTHONDONTWRITEBYTECODE: '1'` to the packaged engine's spawn env
- `engine/process_lifetime.py` — added `env["PYTHONDONTWRITEBYTECODE"] = "1"` to the guardian subprocess's allowlisted env
- `.planning/phases/01-packaging-runtime-foundation/01-03-PLAN.md` — Task 2's verify step and `<done>` line updated to `--verify --deep --strict`

## Decisions Made
See `key-decisions` in frontmatter.

## Deviations from Plan
- **Isolation degrade:** dispatched executor twice via `Agent(gsd-executor, isolation="worktree")`; both times the harness forked the new worktree from `origin/HEAD` (stale, predates this branch's local commits — nothing had been pushed), not local branch tip, per a documented upstream limitation (GSD `worktree.base-check`, reason `head-diverged-from-fork`, ref #48 / claude-code#44965). Escalated to the peer orchestrator session; user approved degrading to sequential execution in the current (correctly-based) worktree for this run only. Code edits within that were still delegated to named agents (`react-dev`, `api-backend`) per this project's model-routing policy — the degrade only skipped the outer isolated-worktree wrapper, not the file-edit delegation rule.
- **Scope addition:** the guardian-subprocess fix (`engine/process_lifetime.py`) was not anticipated by the original Task 2 plan text — it was found by actually reproducing the launch, not by re-reading the plan. Included because Task 4's acceptance criteria (`codesign --verify --deep --strict` passing post-launch) cannot pass without it.
- **Non-blocking deviation from "dialog-free" (Task 4, real hardware):** in-place launch showed no Gatekeeper/AMFI dialog as expected, but macOS did show one TCC file-access privacy prompt (most likely a Documents-folder access request, since the build lives under `~/Documents/Projects`; exact wording not captured). This is an OS privacy prompt, unrelated to code signing — not a regression, not something this phase's threat model covers, and it still passed the plan's actual done-criteria (no Gatekeeper/AMFI dialog, `codesign --verify --deep --strict` clean after quitting). Unverified caveat worth flagging forward: TCC grants are keyed to the code signature, so ad-hoc re-signing (this project's signing mode) may cause macOS to re-prompt on every rebuilt copy — worth watching if this becomes annoying in later phases, but out of scope to fix here.

## Issues Encountered
- `engine/mailhub` git submodule was uninitialized in this worktree (`git submodule update --init --recursive` required) and `engine/runtime` was unprovisioned (`python3 tools/provision-runtime.py` required) before any build could run. Both are worktree-local setup, not phase bugs.
- `npm run build` touched `package-lock.json` (peer-flag normalization noise from local npm version) on one build attempt, which tripped `package-preflight.mjs`'s clean-tree check; discarded via `git checkout -- package-lock.json` before rebuilding — not a source change, nothing to commit.

## User Setup Required
None beyond what Task 4 already required (real Apple Silicon hardware; Intel hardware optional).

## Next Phase Readiness
Phase 1's ROADMAP goal is met: an ad-hoc-signed macOS `.app` builds, launches with no signing-related dialog, survives a real launch/quit cycle with its seal intact, and shows the correct recoverable Gatekeeper dialog when quarantined — verified on Apple Silicon. PKG-01, PKG-02, PKG-03, and RUN-01 are all satisfied. Intel (x64) verification was not run (no Intel hardware available this session) and remains a documented follow-up, per the plan's own allowance for that case. Phase 2 (process lifecycle) can proceed.

### Human verification (completed)
- Task 4's real-hardware Gatekeeper dialog check (D4 above): PASS, run by the user on real Apple Silicon 2026-09-23 — in-place launch dialog-free (bar one unrelated TCC privacy prompt, see Deviations), seal intact after quitting, quarantined copy showed "cannot be verified" not "is damaged".
