---
phase: 01-packaging-runtime-foundation
plan: 01
subsystem: infra
tags: [python-build-standalone, electron-builder, macos, packaging, esbuild]

requires: []
provides:
  - "macOS branch of tools/provision-runtime.py (main_macos) that provisions a real host-arch python-build-standalone runtime under engine/runtime/, verified against astral-sh's own SHA256SUMS"
  - "assertRuntimeLayoutMac in tools/runtime-layout.mjs, the macOS sibling of assertRuntimeLayout"
  - "locateEngineRuntime's darwin default marker (bin/python3.13), and a runtimeDir-driven (not sys.executable-derived) containment root in the shared import probe"
  - "REQUIRED_PACKAGE_INPUTS_MAC + assertPackageInputsPresentMac in tools/preflight-lib.mjs, and platform branches in tools/package-preflight.mjs and tools/stage-runtime.mjs"
affects: [packaging, electron-builder-mac, macos-release]

actuals:
  tokens: 4000
  tasks: 3
  commits: 3

tech-stack:
  added: [python-build-standalone (astral-sh, release 20260901)]
  patterns: ["process.platform branch mirrored across provisioning, layout-assertion, and preflight/staging in lockstep, one darwin counterpart per existing win32 function, Windows path never modified"]

key-files:
  created: []
  modified:
    - tools/provision-runtime.py
    - tools/runtime-layout.mjs
    - tools/preflight-lib.mjs
    - tools/package-preflight.mjs
    - tools/stage-runtime.mjs
    - tests/runtime-layout.test.mjs

key-decisions:
  - "main_macos() downloads and verifies against astral-sh's live SHA256SUMS for the pinned release tag, never a hardcoded hash — mirrors the win32 branch's checksum-then-extract shape but with a fetched, per-release checksum instead of one pinned to the archive."
  - "assertRuntimeImports' containment check now derives its root from the runtimeDir argument passed to it, not sys.executable's immediate parent — the two coincided on Windows (python.exe sits directly in the runtime root) but diverge on macOS (python3.13 sits under bin/), so this is a correctness fix with zero win32 behavior change, needed to make the darwin probe work at all."
  - "stage-runtime.mjs picks assertRuntimeLayout vs assertRuntimeLayoutMac by sniffing which marker (python.exe or bin/python3.13) is actually present on the resolved source/destination directory, rather than trusting process.platform — so runtime:stage keeps working if a mac-provisioned tree is ever staged from/to a non-darwin host."

patterns-established:
  - "Pattern: every Windows-shaped exported function in these packaging/layout modules gets an exact-sibling '*Mac' export with the identical check order and return shape, never a modified original — assertRuntimeLayout/assertRuntimeLayoutMac and assertPackageInputsPresent/assertPackageInputsPresentMac both follow this."

requirements-completed: [RUN-01]

coverage:
  - id: D1
    description: "tools/provision-runtime.py provisions a real, host-arch-matched macOS Python runtime (python-build-standalone) under engine/runtime/, verified against astral-sh's published SHA256SUMS, with dependencies installed and smoke-imported via the runtime's own interpreter"
    requirement: "RUN-01"
    verification:
      - kind: other
        ref: "python3 tools/provision-runtime.py (manual run on this macOS host) — produced engine/runtime/bin/python3.13, engine/runtime/lib/python3.13/site-packages, engine/runtime/runtime-manifest.json; the script's own subprocess smoke-import (fastapi, uvicorn, websockets, httpx, PIL, psutil, engine, mailhub) exited 0"
        status: pass
    human_judgment: false
  - id: D2
    description: "assertRuntimeLayoutMac validates the macOS runtime tree layout (bin/python3.13, lib/python3.13/site-packages, manifest-vs-dist-info completeness, orgtree.pth targets), and locateEngineRuntime defaults to finding a real provisioned runtime on darwin"
    requirement: "RUN-01"
    verification:
      - kind: unit
        ref: "tests/runtime-layout.test.mjs#a correct macOS provisioned layout passes and reports every dependency"
        status: pass
      - kind: unit
        ref: "tests/runtime-layout.test.mjs#assertRuntimeLayoutMac refuses a tree with no bin/python3.13 interpreter, by name"
        status: pass
      - kind: unit
        ref: "tests/runtime-layout.test.mjs#assertRuntimeLayoutMac refuses a manifest dependency with no matching dist-info"
        status: pass
      - kind: unit
        ref: "tests/runtime-layout.test.mjs#locateEngineRuntime's default marker is bin/python3.13 on darwin, python.exe elsewhere"
        status: pass
      - kind: unit
        ref: "tests/runtime-layout.test.mjs#the embedded interpreter imports the representative backend dependencies from inside the runtime (real probe against the Task 1 runtime, on darwin now via assertRuntimeLayoutMac)"
        status: pass
    human_judgment: false
  - id: D3
    description: "package-preflight.mjs and stage-runtime.mjs branch by platform to validate/move a macOS-shaped runtime, while every existing Windows check and test stays byte-for-byte unchanged"
    requirement: "RUN-01"
    verification:
      - kind: automated_ui
        ref: "node tools/package-preflight.mjs — assertPackageInputsPresentMac + assertRuntimeLayoutMac both passed against the Task 1 provisioned runtime (run then hit the unrelated, expected assertReleaseProvenance dirty-tree refusal in this in-progress dev worktree)"
        status: pass
      - kind: unit
        ref: "tests/dev-install.test.mjs (all 9 existing Windows-focused tests)"
        status: pass
      - kind: unit
        ref: "tests/runtime-layout.test.mjs (24/24, full suite including pre-existing Windows cases)"
        status: pass
    human_judgment: false

duration: 55min
completed: 2026-09-17
status: complete
---

# Phase 1 Plan 01: macOS Runtime Provisioning and Package Layout Guardrails Summary

**A real python-build-standalone macOS runtime now provisions cleanly under engine/runtime/, and the existing Windows packaging guardrails (layout assertion, preflight, staging) each gained an exact macOS sibling instead of silently passing or hard-failing on a mac-shaped tree.**

## Performance

- **Tasks:** 3 completed
- **Files modified:** 6

## Accomplishments
- `tools/provision-runtime.py` gained a `main_macos()` branch that downloads the pinned `python-build-standalone` release for the host (or `ORGTREE_RUNTIME_ARCH`-selected) arch, verifies it against astral-sh's own published `SHA256SUMS`, extracts it, installs backend dependencies, writes `runtime-manifest.json` and `orgtree.pth`, and smoke-imports everything through the runtime's own interpreter — verified end-to-end on this host, producing a real working runtime.
- `tools/runtime-layout.mjs` gained `assertRuntimeLayoutMac` as an exact sibling of `assertRuntimeLayout` (same check order, same return shape, Windows function untouched), plus a darwin-aware default marker in `locateEngineRuntime` and a `runtimeDir`-driven (rather than `sys.executable`-derived) containment root in the shared import probe — the latter a correctness fix needed because the interpreter sits under `bin/` on macOS but directly in the runtime root on Windows.
- `tools/preflight-lib.mjs`, `tools/package-preflight.mjs`, and `tools/stage-runtime.mjs` now branch by platform (or by which marker file is actually present, for staging) to validate or move a macOS-shaped runtime, with the Windows pair of functions/checks left completely unmodified.

## Task Commits

1. **Task 1: Provision a macOS Python runtime via python-build-standalone (RUN-01)** - `a0c2cdd` (feat)
2. **Task 2: macOS runtime layout assertion (RUN-01)** - `75158d3` (feat)
3. **Task 3: Platform-aware preflight and staging (RUN-01)** - `401921e` (feat)

## Files Created/Modified
- `tools/provision-runtime.py` - macOS branch (`main_macos`, `_darwin_arch`, `_darwin_verified_download`), dispatched from `main()` before the win32 guard
- `tools/runtime-layout.mjs` - `assertRuntimeLayoutMac` export; darwin-aware `locateEngineRuntime` default marker; `assertRuntimeImports`' interpreter path and containment root made platform-correct
- `tools/preflight-lib.mjs` - `REQUIRED_PACKAGE_INPUTS_MAC` + `assertPackageInputsPresentMac` exports
- `tools/package-preflight.mjs` - `process.platform` branch calling the matching assert pair
- `tools/stage-runtime.mjs` - source/destination marker sniffing (`python.exe` vs `bin/python3.13`) selects the matching layout assertion
- `tests/runtime-layout.test.mjs` - new coverage for `assertRuntimeLayoutMac` (happy path, missing interpreter, incomplete manifest) and the darwin default marker; `fixtureRoot()` now realpath-resolves its tmpdir; the real-runtime import-probe test is platform-aware

## Decisions Made
See `key-decisions` in frontmatter: the fetched (never hardcoded) per-release checksum, the `runtimeDir`-driven containment root in the shared import probe, and marker-sniffing (not `process.platform`) in `stage-runtime.mjs`.

## Deviations from Plan

### Auto-fixed Issues

**1. Latent macOS-only test-fixture bug surfaced by the darwin default marker — `fixtureRoot()` didn't resolve through /var's symlink to /private/var**
- **Found during:** Task 2 verification (`node --test tests/runtime-layout.test.mjs`)
- **Issue:** macOS's `os.tmpdir()` sits behind `/var -> /private/var`. `stageRuntime`'s junction/symlink guard (by design) refuses any ancestor symlink, so every test fixture built on a raw `mkdtempSync()` path failed that guard on this host — a bug invisible on Windows/Linux, never exercised on a real Mac dev host before this plan.
- **Fix:** `fixtureRoot()` now returns `fs.realpathSync(fs.mkdtempSync(...))`. The security check itself was left untouched (weakening it was not an option).
- **Files modified:** `tests/runtime-layout.test.mjs`
- **Verification:** the two previously-failing tests (`runtime staging refuses a source with the RC4 layout...`, plus the general suite) pass.
- **Committed in:** `75158d3` (part of Task 2 commit)

**2. `locateEngineRuntime`'s new darwin default surfaced a real provisioned runtime that the existing "embedded interpreter imports" test then validated with the Windows-only `assertRuntimeLayout`/hardcoded `python.exe` probe**
- **Found during:** Task 2 verification, on this host where Task 1's real runtime now exists
- **Issue:** Before this plan, that test's default-marker lookup never found anything on macOS (no mac runtime shape ever existed), so it always skipped. My correctly-implemented marker default change makes it find the Task 1 runtime, exposing that the test's real-runtime path was hardcoded to Windows assumptions.
- **Fix:** the test now calls `assertRuntimeLayoutMac` on darwin (else `assertRuntimeLayout`, unchanged); `assertRuntimeImports`' interpreter path resolution and the shared probe's containment root were made platform-correct (see key-decisions) so the real import probe actually passes against a mac runtime.
- **Files modified:** `tools/runtime-layout.mjs`, `tests/runtime-layout.test.mjs`
- **Verification:** `node --test tests/runtime-layout.test.mjs` — the test now runs the REAL probe against the Task 1 runtime and passes (previously would have hard-failed instead of skipping, once a real mac runtime exists in the checkout).
- **Committed in:** `75158d3` (part of Task 2 commit)

**3. A Windows-shaped test fixture stub collided with Task 3's new marker-sniffing in `stage-runtime.mjs`**
- **Found during:** Task 3 verification (`node --test tests/dev-install.test.mjs tests/runtime-layout.test.mjs`)
- **Issue:** Fix #2 above had added a `bin/python3.13` stub to the shared `correctRuntime()` (Windows-shaped) fixture so one darwin-affected test could pass. Task 3's `stage-runtime.mjs` then sniffs `bin/python3.13` to pick the mac assertion, so it misdetected that Windows fixture as macOS-shaped in the unrelated RC4-mis-staging test.
- **Fix:** moved the `bin/python3.13` stub out of the shared `correctRuntime()` helper and into the one test that actually needs it (`the import probe fails closed...`).
- **Files modified:** `tests/runtime-layout.test.mjs`
- **Verification:** `node --test tests/dev-install.test.mjs tests/runtime-layout.test.mjs` — 24/24 pass.
- **Committed in:** `401921e` (part of Task 3 commit)

---

**Total deviations:** 3 auto-fixed (all Rule: pre-existing/newly-surfaced correctness gaps in already-in-scope files, required to make each task's own stated verify command pass on this real macOS host).
**Impact on plan:** All three were necessary for correctness once a real macOS runtime and a real macOS host were both present for the first time; none touched a Windows-only code path, and every pre-existing Windows test's behavior is unchanged. No scope creep beyond what each task's own verify step required.

## Issues Encountered
This worktree is shared with a concurrent agent executing a sibling plan (icon generation, `01-02`) in the same directory. Two commit attempts (`git add <file>; git commit`) picked up that agent's concurrently-staged files before the commit landed; both were caught via `git diff --cached --stat` before committing was retried, or fixed after the fact with `git reset --soft`/`git reset` to unstage and recommit with only the intended file(s). Final commits (`a0c2cdd`, `75158d3`, `401921e`) each touch only their own task's files, confirmed via `git show --stat`.

`engine/mailhub` was an uninitialized submodule at the start of this session (`git submodule status` showed a leading `-`); ran `git submodule update --init -- engine/mailhub` to unblock Task 1's smoke-import verification (not a code change, no commit).

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
`engine/runtime/` on this host is now a real, layout-correct macOS Python runtime, and the packaging guardrails understand its shape. The macOS packaging plan's `npm run package:mac:dir` build (icon, electron-builder mac target, ad-hoc signing — tracked separately, in-flight concurrently as plan 01-02) can now be verified against a real runtime instead of a stub.

---
*Phase: 01-packaging-runtime-foundation*
*Completed: 2026-09-17*
