---
phase: 01-packaging-runtime-foundation
plan: 02
subsystem: packaging
tags: [electron-builder, macos, icns, iconutil, icon-assets]

requires:
  - phase: 01-packaging-runtime-foundation (plan 01)
    provides: existing tools/generate-icon.mjs PNG rasterizer and orgtree-eye.ico assets
provides:
  - apps/desktop/assets/orgtree-eye.icns, generated from the existing eye artwork via iconutil
  - tools/generate-icon.mjs writeIcns() function, extending the existing ICO writer to also emit a macOS iconset/icns
  - test coverage proving the .icns exists and round-trips through iconutil
affects: [01-03 (electron-builder mac target config, which will reference this .icns path)]

actuals:
  tokens: 1400
  tasks: 2
  commits: 2

tech-stack:
  added: []
  patterns: ["iconutil-based .iconset -> .icns packaging, reusing the existing dependency-free PNG rasterizer"]

key-files:
  created:
    - apps/desktop/assets/orgtree-eye.icns
  modified:
    - tools/generate-icon.mjs
    - tests/icon-assets.test.mjs
    - .gitignore

key-decisions:
  - "Built the '@2x' retina filename suffix from String.fromCharCode(64) instead of a literal string containing '@' next to a dotted suffix, in both tools/generate-icon.mjs and tests/icon-assets.test.mjs, because this repo's ctx_edit tooling silently redacts any literal text matching an email-like name@domain.tld pattern in written source — which collapsed all four retina iconset filenames into a single '[email protected]' placeholder file on disk the first time they were written as plain string literals, corrupting the .icns (only 6 of 10 required images were actually packed)."
  - "After every generate-icon.mjs run, restored the 7 pre-existing .ico files (orgtree-eye.ico + 6 tray variants) via git checkout before committing: this machine's zlib/Node produces different (but pixel-identical) deflate bytes than whatever generated the committed .ico files, so re-running the untouched ICO code path changes their bytes even though no ICO logic was edited. Only the new .icns was meant to change in this plan."

patterns-established:
  - "macOS .icns generation from the same in-repo PNG rasterizer used for .ico, via iconutil -c icns on a generated .iconset/ directory"

requirements-completed: [PKG-02]

coverage:
  - id: D1
    description: "apps/desktop/assets/orgtree-eye.icns exists, is a valid Mac OS X icon container, and is committed like the existing .ico assets"
    requirement: "PKG-02"
    verification:
      - kind: unit
        ref: "tests/icon-assets.test.mjs#macOS ICNS exists and is non-empty"
        status: pass
      - kind: other
        ref: "file apps/desktop/assets/orgtree-eye.icns | grep -qi 'Mac OS X icon'"
        status: pass
    human_judgment: false
  - id: D2
    description: "The .icns round-trips through iconutil -c iconset back to the full 10-image required size set"
    requirement: "PKG-02"
    verification:
      - kind: unit
        ref: "tests/icon-assets.test.mjs#macOS ICNS round-trips through iconutil to the full 10-image iconset"
        status: pass
      - kind: other
        ref: "iconutil -c iconset -o /tmp/orgtree-icns-roundtrip.iconset apps/desktop/assets/orgtree-eye.icns"
        status: pass
    human_judgment: false
  - id: D3
    description: "The icon actually renders correctly (not distorted/blank) in the macOS Dock and Finder"
    verification: []
    human_judgment: true
    rationale: "Rendering correctness in the Dock/Finder can only be confirmed by a human looking at a built .app on real macOS; this sandbox can only prove the .icns container is structurally valid and contains the right PNG frames, not that it visually renders as the intended eye icon."

duration: 25min
completed: 2026-09-17
status: complete
---

# Phase 01-02: Generate orgtree-eye.icns via iconutil Summary

**Extended the existing dependency-free icon rasterizer to also emit a real macOS `.icns` via `iconutil`, closing the gap that blocked electron-builder's `mac` target.**

## Performance

- **Tasks:** 2 completed
- **Files modified:** 4 (tools/generate-icon.mjs, apps/desktop/assets/orgtree-eye.icns, .gitignore, tests/icon-assets.test.mjs)

## Accomplishments
- `tools/generate-icon.mjs` now writes 10 Apple Icon Image PNG frames into `apps/desktop/assets/orgtree-eye.iconset/` (reusing the existing `png()` rasterizer with the same orange/pupil colors as `orgtree-eye.ico`) and shells out to `iconutil -c icns` to produce `apps/desktop/assets/orgtree-eye.icns`, committed to the repo.
- `apps/desktop/assets/orgtree-eye.iconset/` is gitignored as an intermediate build artifact.
- Two new tests in `tests/icon-assets.test.mjs` assert the `.icns` exists/is non-empty and that it round-trips through `iconutil -c iconset` back to all 10 required PNG filenames.

## Task Commits

1. **Task 1: Generate orgtree-eye.icns via iconutil (PKG-02)** - `0cd22e3` (feat)
2. **Task 2: Test coverage** - `40eac80` (test)

## Files Created/Modified
- `tools/generate-icon.mjs` - added `iconsetSizes`, `writeIcns()`, and a call generating `orgtree-eye.icns`
- `apps/desktop/assets/orgtree-eye.icns` - new, generated macOS icon container (committed)
- `.gitignore` - added `apps/desktop/assets/orgtree-eye.iconset/`
- `tests/icon-assets.test.mjs` - added existence/non-empty and iconutil round-trip tests

## Decisions Made
- Constructed the `@2x` retina filename suffix via `String.fromCharCode(64)` instead of a literal string, in both the tool and the test, because this repo's `ctx_edit` write path silently redacts literal `name@domain.tld`-shaped text as an email address — discovered when it collapsed all four retina iconset filenames into one placeholder file, leaving the `.icns` with only 6 of 10 required images on the first attempt. Rewriting the literal to build the `@` at runtime fixed it; verified byte-for-byte via a standalone Node repro before and after.
- Restored the 7 pre-existing `.ico` files via `git checkout` after every `node tools/generate-icon.mjs` run, before committing: this machine's zlib produces different (but equivalent) deflate bytes than the committed `.ico` files even though the ICO-writing code was never touched, and Plan 01-02 only scopes the `.icns` as a new artifact.

## Deviations from Plan

### Auto-fixed Issues

**1. Silent filename collision in generated iconset (discovered during Task 1)**
- **Found during:** Task 1 (generate orgtree-eye.icns via iconutil)
- **Issue:** Writing the iconset PNG filenames as plain string literals (e.g. `'[email protected]'`) caused the repo's `ctx_edit` tooling to redact them as email-like text, collapsing all four `@2x` filenames into a single `'[email protected]'` file on disk. The resulting `.icns` only contained 6 of the required 10 PNG frames (confirmed by parsing its `icns` chunk table directly).
- **Fix:** Rebuilt the retina filename via `String.fromCharCode(64) + '2x.png'` in `tools/generate-icon.mjs` and the mirrored expected-filename list in `tests/icon-assets.test.mjs`, avoiding any literal `@` character adjacent to a dotted suffix in source.
- **Files modified:** `tools/generate-icon.mjs`, `tests/icon-assets.test.mjs`
- **Verification:** Re-ran `node tools/generate-icon.mjs`; confirmed all 10 distinct filenames on disk in the iconset dir and all 10 recovered by `iconutil -c iconset` round-trip.
- **Committed in:** `0cd22e3` (Task 1) and `40eac80` (Task 2)

---

**Total deviations:** 1 auto-fixed (environment/tooling gotcha, not a plan error)
**Impact on plan:** No scope creep — fix was required for the `.icns` to be structurally valid at all.

## Issues Encountered
- This worktree had a concurrent session actively committing unrelated work (plan 01.01-01, `tools/provision-runtime.py`, RUN-01) to the same branch while this plan executed. A first `git commit` attempt for Task 1 landed a merged commit (`154f6bf`) mixing that unrelated staged file into this plan's commit under a foreign message — an existing safety hook detected the mismatch against this plan's `files_modified` and auto-reset it out before it could be reported. Both of this plan's commits were then re-done with explicit pathspecs (`git commit -- <files>`) and verified via `git show --stat` to contain only the intended files.

## User Setup Required
None - no external service configuration required.

## Next Phase Readiness
`apps/desktop/assets/orgtree-eye.icns` exists at the exact path Plan 01-03's `build.mac.icon` config is expected to reference. Human verification of actual Dock/Finder rendering is still required (see below) before this can be considered visually correct, not just structurally valid.

### Human verification needed (cannot be completed by this agent)
Build the macOS `.app` (e.g. `npm run` whatever packaging script invokes electron-builder's `mac` target, or `electron-builder --mac`) and visually inspect:
1. The app icon in the Dock while running.
2. The app icon in Finder (the `.app` bundle icon, at both small list-view and large icon-view sizes).
3. Confirm it shows the orange eye artwork clearly, un-distorted, at each size — not a blank/default Electron icon or a blurry/pixelated one.

This was not marked done above (`D3`, `human_judgment: true`) since no human-observed screenshot or build was available in this session.

---
*Phase: 01-packaging-runtime-foundation*
*Completed: 2026-09-17*
