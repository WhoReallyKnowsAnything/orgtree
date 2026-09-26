# Phase 1: Packaging & Runtime Foundation - Research

**Researched:** 2026-09-17
**Domain:** electron-builder macOS packaging, ad-hoc code signing, python-build-standalone runtime bundling, `.icns` icon generation
**Confidence:** MEDIUM

## Summary

Phase 1 is additive packaging work on top of an existing, working Windows build — not a new architecture. `electron-builder` 26.15.3 (already pinned) supports a `mac` target out of the box; no new npm dependency is needed. The two real risks are (1) Apple Silicon's kernel-level AMFI signature enforcement, which blocks *any* unsigned Mach-O binary at launch even with zero network involvement, and (2) the project's existing package-layout guardrails (`tools/preflight-lib.mjs`, `tools/runtime-layout.mjs`) that hard-code the Windows embeddable-Python layout (`python.exe`, `python313.zip`, `python313._pth`) and will reject or silently bypass a macOS build unless given a platform-specific counterpart.

python-build-standalone's official release for the exact CPython version already pinned in `tools/provision-runtime.py` (**3.13.15**, not 3.10.x — see the `Assumptions Log` correction below) ships `aarch64-apple-darwin` and `x86_64-apple-darwin` `install_only` tarballs with a **completely different layout** than the Windows embeddable zip: `bin/python3.13`, `lib/python3.13/site-packages`, `lib/python3.13/lib-dynload/*.so`, and loose `lib/*.dylib` files — no `._pth` file, no zip-archived stdlib. `provision-runtime.py`'s macOS branch and the layout-assertion helpers both need this distinction, not a reuse of the Windows checks.

For signing: electron-builder's own mac signing pass (triggered by `mac.identity: "-"`) correctly signs the `.app` bundle and its known `Frameworks`/dylibs — manual `codesign --force --deep` is unnecessary and actively discouraged by Apple DTS. But it does **not** automatically sign arbitrary binaries dropped in via `extraResources` (i.e., the bundled Python runtime). Those must be explicitly signed, either via the `mac.binaries` config list (no glob support — impractical for python-build-standalone's ~10+ `.so`/`.dylib` files) or, more practically, via a custom `afterPack` hook that walks the staged runtime directory and ad-hoc-signs every Mach-O file *before* electron-builder's own signing phase runs.

**Primary recommendation:** Add `mac.target: [{target: "dmg", arch: ["arm64","x64"]}, {target: "zip", arch: ["arm64","x64"]}]`, `mac.identity: "-"`, `mac.hardenedRuntime: false` (simplest correct choice for a local, unnotarized build — avoids needing an entitlements file at all), `mac.icon: "apps/desktop/assets/orgtree-eye.icns"`, and an `afterPack` hook that ad-hoc-signs every file under the staged `engine/runtime/` tree before packaging proceeds to electron-builder's own sign phase.

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| macOS app bundling (.app/.dmg/.zip) | Build tooling (electron-builder config in `package.json`) | — | electron-builder already owns Windows NSIS packaging; the `mac` target is config, not new code |
| Ad-hoc code signing (top-level + nested) | Build tooling (electron-builder `mac.identity`) | Custom `afterPack` hook (Node, `tools/`) | electron-builder signs its own known bundle structure; loose `extraResources` binaries need an explicit pre-sign pass owned by project build scripts |
| Python runtime provisioning (macOS) | Build tooling (`tools/provision-runtime.py`, Python) | — | Direct macOS analog of the existing Windows branch in the same file; build-time only, never a runtime component |
| Runtime layout verification | Build tooling (`tools/runtime-layout.mjs`, `tools/preflight-lib.mjs`) | — | Existing Windows-only layout assertions must gain a macOS-specific counterpart; this is packaging-time verification, not app runtime code |
| Icon asset generation (.icns) | Build tooling (`tools/generate-icon.mjs` extension + `iconutil`) | OS (macOS `iconutil` binary) | Existing hand-rolled PNG rasterizer already in-repo; `iconutil` itself is an Apple CLI, not something to reimplement |
| First-launch Gatekeeper/AMFI approval (PKG-03) | OS (macOS Gatekeeper/AMFI) | Documentation (project docs) | No code fix exists for this — it's an OS-level, per-machine trust decision; only user-facing docs + a manual hardware verification step are in scope |

## Standard Stack

### Core

| Library | Version | Purpose | Why Standard |
|---------|---------|---------|--------------|
| electron-builder | 26.15.3 (pinned, unchanged) | Produces `.app`/`.dmg`/`.zip` for macOS, same tool already producing the Windows NSIS installer | `[VERIFIED: npm registry — npm view electron-builder version → 26.15.3, published 2026-09-07]`. Adding a `mac` target reuses the existing single build tool rather than introducing Electron Forge or a second packager. |
| python-build-standalone (astral-sh org) | release tag `20260901`, CPython `3.13.15` | Relocatable macOS Python interpreter bundled under `engine/runtime/` | `[VERIFIED: github.com/astral-sh/python-build-standalone releases API — asset names cpython-3.13.15+20260901-aarch64-apple-darwin-install_only.tar.gz and cpython-3.13.15+20260901-x86_64-apple-darwin-install_only.tar.gz confirmed present]`. Version **3.13.15** was chosen to match the exact CPython version already hardcoded for the Windows runtime — see `[VERIFIED: tools/provision-runtime.py:15]` `VERSION = "3.13.15"`. |

No new npm or pip packages are introduced by this phase — see `Package Legitimacy Audit` below.

### Supporting

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `iconutil` (Apple, bundled with macOS) | n/a — OS binary at `/usr/bin/iconutil` | Converts a `.iconset` folder of PNGs into a `.icns` | `[VERIFIED: local \`man iconutil\` on Darwin 27.2.0 — synopsis "iconutil -c {icns \| iconset} [-o file] file"]`. Confirmed present: `[VERIFIED: command -v iconutil → /usr/bin/iconutil]`. |
| `codesign` (Apple, bundled with macOS) | n/a — OS binary at `/usr/bin/codesign` | Ad-hoc-signs the bundled Python runtime binaries before electron-builder's own sign pass | `[VERIFIED: command -v codesign → /usr/bin/codesign]` |
| existing `tools/generate-icon.mjs` PNG writer (`png()` function) | n/a, in-repo | Reuse the existing dependency-free PNG rasterizer to emit `.iconset` PNGs at required sizes from the same eye-vector source used for the `.ico` | `[VERIFIED: tools/generate-icon.mjs signatures — exports insideEye(), pixel(), crc32(), chunk(), png(), writeIcon()]`. No new image library (sharp/jimp) needed — the eye shape is defined as parametric cubic-bezier coordinates, not a raster source, so it rasterizes cleanly at the larger 512/1024px sizes `.icns` requires. |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| electron-builder `mac` target | Electron Forge, electron-packager | Would introduce a second packaging tool/config for one platform; no benefit since electron-builder already fully supports macOS |
| `afterPack` hook signing extraResources | List every runtime binary explicitly in `mac.binaries` | `mac.binaries` has no glob support `[CITED: github.com/electron-userland/electron-builder MacTargetHelper.ts — "There is no glob expansion anywhere in this pipeline — only explicit paths are accepted"]`; python-build-standalone ships 7+ `.dylib` and multiple `.so` files whose exact list depends on the release build, making an explicit list brittle |
| `mac.hardenedRuntime: false` | Keep `hardenedRuntime: true` + add `entitlements.mac.plist` with `com.apple.security.cs.disable-library-validation` and `com.apple.security.cs.allow-jit` | Keeping hardened runtime on preserves more OS protections but adds an entitlements file and two required entries to get right; for a purely local, unnotarized build the security benefit is minimal since hardened runtime mainly matters for notarized distribution |

**Installation:** No new packages to install — `electron-builder` is already a devDependency; `iconutil`/`codesign` are OS-bundled.

**Version verification:** `[VERIFIED: npm registry]` `npm view electron-builder version` → `26.15.3` (already pinned in `package.json`, matches). `npm view electron-builder versions` confirms `26.15.3` predates the `27.0.0-alpha.*` line, so the **flat** `mac.identity`/`mac.hardenedRuntime`/`mac.binaries` config keys apply — the `mac.sign.*` nested form is a v27 breaking change not yet released as stable and does not apply to this pin `[CITED: github.com/electron-userland/electron-builder/website/docs/migration/v27-breaking-changes.md]`.

## Package Legitimacy Audit

No new npm, pip, or cargo packages are installed in this phase.

- `electron-builder` — already a devDependency, unchanged version.
- python-build-standalone — not a package-registry install; it is a raw release-tarball download (like the existing Windows `python-{VERSION}-embed-amd64.zip` fetch in `tools/provision-runtime.py`) verified by SHA256, not `npm install`/`pip install`. The macOS branch should follow the same pattern: download over HTTPS, verify a checksum before extracting (the existing Windows code already does this for its zip — `[VERIFIED: tools/provision-runtime.py]` `if hashlib.sha256(archive.read_bytes()).hexdigest() != SHA256: raise SystemExit(...)`).

**Packages removed due to [SLOP] verdict:** none
**Packages flagged as suspicious [SUS]:** none

## Architecture Patterns

### System Architecture Diagram

```
npm run package:mac (new script, mirrors package:win)
        │
        ▼
tools/build.mjs (existing, platform-agnostic bundling of main/preload/renderer)
        │
        ▼
tools/package-preflight.mjs
        │  assertPackageInputsPresent() ─── needs a macOS branch:
        │      engine/runtime/bin/python3.13 (not python.exe)
        │      engine/runtime/lib/python3.13/site-packages (not Lib/site-packages)
        │  assertRuntimeLayout() ─── needs a macOS-specific layout checker
        ▼
electron-builder (mac target)
        │
        ├─ afterPack hook (NEW) ── ad-hoc signs every Mach-O binary under
        │                          staged engine/runtime/** BEFORE electron-builder's
        │                          own signing phase runs
        │
        ▼
electron-builder's own "Phase 2g — Code Sign"
        │  signs Contents/MacOS/*, Contents/Frameworks/**, top-level .app
        │  (mac.identity: "-" → ad-hoc; does NOT re-walk Contents/Resources)
        ▼
dmg + zip artifacts in release/
        │
        ▼
First launch on real hardware (manual, PKG-03)
        │  AMFI checks every Mach-O's signature at load time (Apple Silicon)
        │  Gatekeeper/quarantine check only applies if the .app carries
        │  com.apple.quarantine (e.g. after AirDrop/zip transfer, not a
        │  same-machine local build)
```

### Recommended Project Structure

No new top-level directories. Additions live inside existing conventions:

```
apps/desktop/assets/
├── orgtree-eye.ico          # existing, Windows — unchanged
├── orgtree-eye.icns         # NEW — generated via iconutil, referenced by build.mac.icon
└── orgtree-eye.iconset/     # NEW — intermediate build artifact, gitignored, regenerated each build

tools/
├── generate-icon.mjs        # EXTEND — add an iconset-PNG export path alongside the existing ICO path
├── provision-runtime.py     # EXTEND — add a macOS branch (python-build-standalone), keep the win32 branch untouched
├── runtime-layout.mjs       # EXTEND — add a macOS-specific layout assertion (different file set than Windows)
└── sign-runtime-macos.mjs   # NEW — the afterPack hook implementation, invoked from package.json build.afterPack
```

### Pattern 1: Pre-sign extraResources binaries in `afterPack`, before electron-builder's own sign phase

**What:** electron-builder's built-in signing only covers the bundle structure it manages (`Contents/MacOS`, `Contents/Frameworks`). Anything staged via `extraResources` (here, `engine/runtime/`) lands under `Contents/Resources/` and is not walked by the automatic signer.
**When to use:** Any time a build bundles a foreign, unsigned binary (interpreter, native helper) via `extraResources`.
**Example:**
```javascript
// package.json → build.afterPack: "tools/sign-runtime-macos.mjs"
// Source: pattern derived from electron-builder hook docs
// (github.com/electron-userland/electron-builder/website/docs/features/hooks.md —
//  "afterPack: After files are packaged, before signing")
import { execFileSync } from 'node:child_process'
import fs from 'node:fs'
import path from 'node:path'

export default async function afterPack(context) {
  if (context.electronPlatformName !== 'darwin') return
  const runtimeDir = path.join(context.appOutDir, `${context.packager.appInfo.productFilename}.app`,
    'Contents', 'Resources', 'engine', 'runtime')
  if (!fs.existsSync(runtimeDir)) return
  for (const file of walk(runtimeDir)) {
    // ad-hoc sign every regular file; codesign no-ops/errors harmlessly on
    // non-Mach-O files (skip failures rather than treat them as fatal)
    try {
      execFileSync('codesign', ['--force', '--sign', '-', file])
    } catch { /* not a signable Mach-O file (e.g. a .py or .txt) */ }
  }
}

function* walk(dir) {
  for (const entry of fs.readdirSync(dir, { withFileTypes: true })) {
    const full = path.join(dir, entry.name)
    if (entry.isDirectory()) yield* walk(full)
    else yield full
  }
}
```

### Pattern 2: Ad-hoc signing config for a local, unnotarized build

**What:** Skip real certificate signing entirely; use electron-builder's ad-hoc sentinel plus a hardened-runtime choice that avoids needing an entitlements file.
**When to use:** MAC-01 scope — no Apple Developer account, local build only.
```json5
// package.json → build.mac (electron-builder 26.15.3, FLAT keys — not mac.sign.*)
// Source: github.com/electron-userland/electron-builder/website/docs/features/code-signing/code-signing-mac.md
// and website/docs/migration/v27-breaking-changes.md (confirms flat keys are pre-v27)
{
  "mac": {
    "identity": "-",
    "hardenedRuntime": false,
    "icon": "apps/desktop/assets/orgtree-eye.icns",
    "target": [
      { "target": "dmg", "arch": ["arm64", "x64"] },
      { "target": "zip", "arch": ["arm64", "x64"] }
    ]
  },
  "afterPack": "tools/sign-runtime-macos.mjs"
}
```

### Pattern 3: `.iconset` → `.icns` via `iconutil`

**What:** Generate the 10 required PNG sizes from the existing vector eye definition, write them into a `.iconset` folder, shell out to `iconutil`.
```javascript
// Source: local `man iconutil` (Darwin 27.2.0) + Apple Icon Image format
// required sizes (en.wikipedia.org/wiki/Apple_Icon_Image_format)
import { execFileSync } from 'node:child_process'
import fs from 'node:fs'
// reuse png(), pixel(), insideEye() already exported by tools/generate-icon.mjs
const iconsetSizes = [
  ['icon_16x16.png', 16], ['[email protected]', 32],
  ['icon_32x32.png', 32], ['[email protected]', 64],
  ['icon_128x128.png', 128], ['[email protected]', 256],
  ['icon_256x256.png', 256], ['[email protected]', 512],
  ['icon_512x512.png', 512], ['[email protected]', 1024],
]
// ... write each PNG into orgtree-eye.iconset/ using the existing png() writer ...
execFileSync('iconutil', ['-c', 'icns', 'apps/desktop/assets/orgtree-eye.iconset',
  '-o', 'apps/desktop/assets/orgtree-eye.icns'])
```

### Anti-Patterns to Avoid

- **`codesign --force --deep -s - MyApp.app` after electron-builder already signed it:** Apple DTS explicitly recommends against `--deep` `[CITED: WebSearch summary of Apple Developer Forums / electron/osx-sign discussion]` — it can re-sign nested items out of order or skip ones electron-builder already correctly handled. Let electron-builder's own signing phase (triggered by `mac.identity`) do the top-level/Frameworks work; only hand-sign what electron-builder doesn't reach (`extraResources`).
- **Assuming the Windows `runtime-layout.mjs`/`preflight-lib.mjs` checks "just work" on macOS:** They hard-code `python.exe`, `python313.zip`, `python313._pth`, `Lib/site-packages` `[VERIFIED: tools/preflight-lib.mjs]` `REQUIRED_PACKAGE_INPUTS = [..., 'engine/runtime/python.exe', 'engine/runtime/python313.zip', ...]` and `[VERIFIED: tools/runtime-layout.mjs:124-148]` `requireFile(path.join(runtimeDir, 'python.exe'), ...)`, `requireFile(path.join(runtimeDir, 'python313.zip'), ...)`, `requireDir(path.join(runtimeDir, 'Lib', 'site-packages'), ...)`. None of these paths exist in a python-build-standalone macOS layout (`bin/python3.13`, `lib/python3.13/site-packages`). A macOS build will either fail these checks outright or (worse) silently skip them if not wired in — this needs an explicit macOS branch, not a shared cross-platform check.
- **Treating PKG-03's Gatekeeper/AMFI flow as one single mechanism:** AMFI (kernel-level, checks every Mach-O signature at load regardless of quarantine) and Gatekeeper (checks `com.apple.quarantine`-flagged files) are different systems. A same-machine local build+run never gains a quarantine flag, so only AMFI applies — correct ad-hoc signing of the full bundle is sufficient and no user-facing "right-click → Open" step should be needed for that path. Document the quarantine-triggered flow (AirDrop/zip transfer to another Mac) separately from the local-build flow so PKG-03's docs don't overstate what manual approval is normally required.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| `.iconset` → `.icns` conversion | A custom ICNS binary-format writer | `iconutil -c icns` (OS-bundled) | ICNS is Apple's own container format with several legacy chunk types; `iconutil` is the canonical, always-correct converter and ships with every macOS install |
| PNG generation for iconset sizes | A new image library dependency (sharp, jimp, canvas) | The existing `png()`/`pixel()`/`insideEye()` functions in `tools/generate-icon.mjs` | Already in-repo, dependency-free, and the eye shape is vector-defined so it rasterizes correctly at the larger 512/1024px sizes `.icns` needs that the current `.ico` sizes array (max 256) doesn't cover |
| Mach-O binary signing | A hand-rolled Mach-O signature writer/parser | `codesign` (OS-bundled) | Code signing format is complex and security-sensitive; this is exactly the kind of thing that must never be reimplemented |
| macOS/Windows runtime layout differences | One shared cross-platform `assertRuntimeLayout()` with `if (platform === 'darwin')` branches threaded through Windows-specific logic | A separate, parallel macOS layout-assertion function (same module, sibling export) | The layouts share no structural overlap (`._pth`/zip-stdlib vs `bin/`+`lib/`); forcing them through one function increases the risk of the Windows check becoming accidentally optional |

**Key insight:** Everything genuinely new in this phase (signing, icon conversion) is an OS-bundled tool, not a library gap — the only actual coding work is glue: an `afterPack` hook, a platform branch in `provision-runtime.py`, and a parallel layout-assertion function.

## Common Pitfalls

### Pitfall 1: Apple Silicon AMFI blocks unsigned/partially-signed apps even with zero network involvement
**What goes wrong:** A locally-built `.app`, never downloaded, still fails to launch on Apple Silicon with "App is damaged and can't be opened."
**Why it happens:** AMFI is a kernel-level check independent of Gatekeeper/quarantine — every Mach-O must carry at least an ad-hoc signature to execute at all `[CITED: .planning/research/PITFALLS.md, cross-referencing Apple Developer Forums]`.
**How to avoid:** Ensure both electron-builder's own signing pass (`mac.identity: "-"`) AND the `afterPack` pre-sign step (for `extraResources` binaries) run on every build. Verify with `codesign -dv --verbose=4` on the built `.app` and on a sample file inside `Contents/Resources/engine/runtime/`.
**Warning signs:** The "damaged" dialog on Apple Silicon but the identical build launches fine on Intel (Intel's Gatekeeper is quarantine-gated; Apple Silicon's AMFI is not).

### Pitfall 2: Package-layout guardrails reject (or worse, don't check) a macOS build
**What goes wrong:** `tools/package-preflight.mjs` calls `assertPackageInputsPresent()` and `assertRuntimeLayout('engine/runtime', ...)`, both of which currently only understand the Windows embeddable-zip layout.
**Why it happens:** These checks were written for one platform and never abstracted `[VERIFIED: tools/preflight-lib.mjs]` (`REQUIRED_PACKAGE_INPUTS` lists `engine/runtime/python.exe`, `engine/runtime/python313.zip` unconditionally) and `[VERIFIED: tools/runtime-layout.mjs]` (`requireFile(path.join(runtimeDir, 'python.exe'), ...)` unconditionally).
**How to avoid:** Add an explicit macOS branch to both functions (or parallel macOS-specific functions) before wiring `npm run package:mac` — do not skip or stub these checks, since the project's own history (`2.1.4-RC4`) shows exactly this class of check catching a real shipped-broken build.
**Warning signs:** `npm run package:mac` "succeeds" but the check never actually ran against the macOS runtime shape (false green).

### Pitfall 3: Bundled Python version mismatch with prior research
**What goes wrong:** Assuming the macOS runtime should target CPython 3.10.x (an earlier research pass's assumption) instead of matching the actual pinned Windows version.
**Why it happens:** `.planning/research/STACK.md` (from initial project research) stated the engine's target as "3.10.13" — but the actual authoritative source, `tools/provision-runtime.py`, hardcodes `VERSION = "3.13.15"` for the Windows runtime `[VERIFIED: tools/provision-runtime.py:15]`. The 3.10.13 figure most likely reflects a local system Python version, not the bundled engine runtime's version.
**How to avoid:** Use `3.13.15` (or the closest available python-build-standalone tag if 3.13.15 itself isn't published — it is, confirmed in this session) for the macOS runtime, so backend dependency wheels resolve consistently on both platforms.
**Warning signs:** `runtime-requirements.in` dependencies failing to install or behaving differently between platforms due to a Python minor-version mismatch.

### Pitfall 4: Confusing local-build AMFI enforcement with quarantine-triggered Gatekeeper flow
**What goes wrong:** Documenting a "right-click → Open" step (PKG-03) as required for every first launch, when it's only needed if the `.app` carries a `com.apple.quarantine` extended attribute (e.g., after zip/AirDrop transfer to a different Mac).
**Why it happens:** Both AMFI and Gatekeeper produce user-visible "won't open" dialogs, but are triggered by different conditions.
**How to avoid:** Test both paths explicitly on real hardware: (1) build and run in-place on the same machine (should just launch, given correct ad-hoc signing), and (2) `zip` the `.app`, transfer it (e.g. via `xattr -w com.apple.quarantine ...` to simulate, or an actual AirDrop/download), and confirm the right-click→Open / System Settings approval flow is what's needed there.
**Warning signs:** PKG-03 documentation describing a manual approval step that testing later shows isn't actually necessary for the primary local-build-and-run flow.

## Code Examples

See `Architecture Patterns` above for the three load-bearing examples (`afterPack` signing hook, `mac` config block, `.iconset`/`iconutil` generation) — each is sourced inline.

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|---------------|--------|
| `mac.identity`, `mac.hardenedRuntime`, `mac.entitlements` as flat top-level `mac.*` keys | `mac.sign.identity`, `mac.sign.hardenedRuntime`, `mac.sign.entitlements` nested under `mac.sign` | v27 (currently `27.0.0-alpha.*` only, `[VERIFIED: npm view electron-builder versions]`) | Not applicable to this phase — project is pinned to `26.15.3`; flat keys are correct today, but a future electron-builder major-version bump will require migrating to `mac.sign.*` |
| `launchctl load`/`unload` | `launchctl bootstrap gui/<uid>`/`bootout gui/<uid>` | macOS 10.10+ | Not used in this phase (Phase 3 concern), noted here since it appeared in prior research and is easy to conflate with Phase 1 |

**Deprecated/outdated:** None directly relevant to Phase 1's scope beyond the v27 signing-key restructure above, which doesn't apply to the pinned version.

## Assumptions Log

| # | Claim | Section | Risk if Wrong |
|---|-------|---------|---------------|
| A1 | `mac.hardenedRuntime: false` (rather than `true` + entitlements file) is the right tradeoff for this local, unnotarized build | Standard Stack / Architecture Patterns | Low — if the team later wants hardened-runtime protections even for local builds, this is a one-line config change plus an entitlements file; nothing else in the plan depends on this choice |
| A2 | electron-builder's `afterPack` hook receives an `appOutDir` whose structure is `<ProductName>.app/Contents/Resources/<extraResources "to" path>` as configured in this project's `extraResources` block | Architecture Patterns, Pattern 1 | Medium — if the exact staged path differs, the `afterPack` hook's `runtimeDir` computation needs adjusting; verify by inspecting `release/mac*/…` after a first `--dir` build before wiring signing |
| A3 | codesign silently errors (non-fatal, catchable) rather than corrupting the file when run against a non-Mach-O regular file (e.g. a stray `.txt` or `.json` under the runtime tree) | Architecture Patterns, Pattern 1 | Low — this is standard `codesign` behavior (it refuses non-code files with an error, doesn't modify them), but should be confirmed by a plan verification step running the hook against a real staged runtime dir |

**Note:** Corrections A1 and the STACK.md Python-version discrepancy (Pitfall 3) are not `[ASSUMED]` — they are `[VERIFIED]` against `tools/provision-runtime.py` directly and supersede the earlier `.planning/research/STACK.md` draft finding.

## Open Questions (RESOLVED)

1. **Does the project want a `dmg` artifact at all for a purely local/dev build, or is `zip`/`--dir` sufficient?** — RESOLVED in 01-03-PLAN.md Task 1: adds both `package:mac:dir` (fast, `electron-builder --mac --dir`) and `package:mac` (dmg+zip), mirroring the existing `package:dir`/`package:win` pair.
   - What we know: `mac.target` defaults to `["zip", "dmg"]` if unspecified; both are cheap to produce.
   - What's unclear: Whether `npm run package:mac` should mirror `package:win` (full installer-style artifact) or `package:dir` (fastest iteration, no compression step) as its primary dev workflow.
   - Recommendation: Add both a `package:mac:dir` (fast, `electron-builder --mac --dir`) and `package:mac` (dmg+zip) script, mirroring the existing `package:dir`/`package:win` pair.

2. **Should the x64 (Intel) build be validated in Phase 1, or deferred given the dev machine is Apple Silicon?** — RESOLVED in 01-03-PLAN.md Task 1 + Task 4: primary build target is arm64-only (avoids the `extraResources` wrong-arch bug); x64 is a documented separate re-provision-then-`package:mac:x64` path, with real-hardware verification deferred to Task 4's checkpoint per PKG-03.
   - What we know: python-build-standalone ships `x86_64-apple-darwin` tarballs; electron-builder can cross-build a `x64` artifact from an `arm64` host.
   - What's unclear: Whether the smoke-import step in `provision-runtime.py` (which must *execute* the target interpreter) can run for a foreign arch without Rosetta 2, and whether Rosetta is installed/available for CI or dev-machine verification.
   - Recommendation: Build both archs, but only require the smoke-import test to pass for the arch matching the build host (`arm64` here); document the x64 path as needing verification on real Intel hardware or via Rosetta, consistent with PKG-03's "verify on real hardware" framing already in REQUIREMENTS.md.

## Environment Availability

| Dependency | Required By | Available | Version | Fallback |
|------------|------------|-----------|---------|----------|
| `iconutil` | PKG-02 | ✓ | OS-bundled (Darwin 27.2.0) | — |
| `codesign` | PKG-01, `afterPack` signing | ✓ | OS-bundled (Darwin 27.2.0) | — |
| `node` | build tooling | ✓ | v26.0.0 | — |
| `npm` | build tooling | ✓ | 12.0.2 | — |
| `python3` (build host) | `tools/provision-runtime.py` | ✓ | 3.10.13 (host system Python — NOT the bundled engine runtime version, see Pitfall 3) | — |
| Apple Silicon (arm64) build host | native arch verification | ✓ | `uname -m` → `arm64` | x64 artifact must be verified separately, see Open Question 2 |
| Xcode Command Line Tools | `codesign`/`iconutil` toolchain | ✓ | `/Applications/Xcode-beta.app/Contents/Developer` | — |

**Missing dependencies with no fallback:** none
**Missing dependencies with fallback:** x64/Intel runtime verification (no Intel hardware confirmed in this environment; fallback is Rosetta 2 or deferred manual verification per Open Question 2)

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | Node's built-in `node:test` (TS/build suite); plain `unittest` (Python backend, not touched by this phase) |
| Config file | none — `node --test tests/*.test.mjs` |
| Quick run command | `node --test tests/icon-assets.test.mjs` |
| Full suite command | `npm test` (→ `node --test tests/*.test.mjs`) |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| PKG-01 | `electron-builder` produces a launchable, ad-hoc-signed macOS `.app` | integration/manual | `npm run package:mac:dir && codesign -dv --verbose=4 release/mac*/Orgtree.app` | ❌ Wave 0 — needs `package:mac:dir` script |
| PKG-02 | `.icns` asset exists and is referenced by `build.mac.icon` | unit | new test mirroring `tests/icon-assets.test.mjs`'s existing ICO pixel/shape assertions, adapted for `.icns` | ❌ Wave 0 |
| PKG-03 | First-launch approval flow documented; manual hardware verification | manual-only | n/a — documentation + a human running the built `.app` on real Apple Silicon hardware | ❌ Wave 0 — no code path can automate an OS trust-prompt interaction |
| RUN-01 | `tools/provision-runtime.py` provisions a macOS runtime instead of hard-exiting | integration | `python3 tools/provision-runtime.py` on a macOS host, then assert the new macOS `assertRuntimeLayout`-equivalent passes | ❌ Wave 0 — needs the new macOS layout-assertion function first |

### Sampling Rate

- **Per task commit:** `node --test tests/icon-assets.test.mjs` (fast, covers icon shape/asset regressions) plus any new unit test added for the macOS layout assertion.
- **Per wave merge:** `npm test` (full existing suite — ensures the Windows path is untouched) plus a manual `npm run package:mac:dir` smoke build.
- **Phase gate:** Full suite green, plus a real on-hardware launch test on Apple Silicon before `/gsd:verify-work` (PKG-03 cannot be closed by automation alone).

### Wave 0 Gaps

- [ ] `package.json` — add `package:mac:dir` and `package:mac` scripts (mirroring `package:win`/`package:dir`)
- [ ] `tools/sign-runtime-macos.mjs` — new `afterPack` hook implementation
- [ ] A macOS-specific layout assertion (new export in `tools/runtime-layout.mjs` or a sibling module) — RUN-01 has no automated verification without this
- [ ] `tests/icns-assets.test.mjs` (or extend `tests/icon-assets.test.mjs`) — asserts the `.icns` file exists and (at minimum) that `iconutil -c iconset` round-trips it back to the expected sizes
- [ ] Framework install: none — `node:test` is already in use, no new framework needed

## Security Domain

### Applicable ASVS Categories

| ASVS Category | Applies | Standard Control |
|---------------|---------|-------------------|
| V2 Authentication | No | Phase 1 has no auth surface — packaging/build tooling only |
| V3 Session Management | No | N/A |
| V4 Access Control | No | N/A |
| V5 Input Validation | No | No user/network input processed by this phase's code (build-time scripts only) |
| V6 Cryptography | Marginal — checksum verification | Reuse the existing `hashlib.sha256` verification pattern already in `tools/provision-runtime.py` for the macOS python-build-standalone download; do not skip checksum verification for the new download path |

### Known Threat Patterns for this stack

| Pattern | STRIDE | Standard Mitigation |
|---------|--------|----------------------|
| Supply-chain tampering of the downloaded python-build-standalone tarball | Tampering | SHA256 checksum verification before extraction, matching the existing Windows-branch pattern in `tools/provision-runtime.py` |
| A malformed/oversized `afterPack` signing loop walking outside the intended `engine/runtime` directory (path traversal into the rest of the bundle) | Tampering | Scope the `afterPack` walk strictly to the resolved `engine/runtime` path under `appOutDir`; do not accept externally-influenced path input (there is none here — paths are all build-time constants) |

## Sources

### Primary (HIGH confidence)
- `[VERIFIED]` `tools/provision-runtime.py` (read directly, full file) — Windows runtime version (`3.13.15`), checksum-verification pattern, dependency-install pattern
- `[VERIFIED]` `tools/preflight-lib.mjs`, `tools/runtime-layout.mjs` (read directly) — existing Windows-only layout guardrails
- `[VERIFIED]` `github.com/astral-sh/python-build-standalone` releases API (tag `20260901`) — exact macOS tarball names, via direct `curl`+`tar -tzf` inspection of the actual archive contents
- `[VERIFIED]` local `man iconutil`, `command -v codesign/iconutil` on Darwin 27.2.0 (this environment)
- `[VERIFIED]` `npm view electron-builder version` / `versions` — 26.15.3 pinned, predates 27.0.0-alpha line

### Secondary (MEDIUM confidence)
- `[CITED: context7 /electron-userland/electron-builder]` — mac target defaults, `mac.binaries` no-glob behavior, build lifecycle hook ordering (`afterPack` before signing), v27 `mac.sign.*` migration table, ad-hoc signing source (`MacTargetHelper.ts`) confirming the `hardenedRuntime` + `disable-library-validation` warning

### Tertiary (LOW confidence)
- `[CITED: electron.build/docs/features/code-signing/code-signing-mac/]` (WebFetch of live docs site, reflects v27 nested syntax — cross-checked against the v26 flat-key migration table above)
- `[ASSUMED]` WebSearch summaries on `codesign --deep` being discouraged by Apple DTS (not independently verified against an Apple-authored primary source in this session)
- Prior `.planning/research/STACK.md`/`PITFALLS.md` findings (already MEDIUM-confidence per that document) — used as background, with one correction noted (Pitfall 3, Python version)

## Metadata

**Confidence breakdown:**
- Standard stack: MEDIUM — electron-builder config verified against source/docs via context7 and version-checked against npm; python-build-standalone tarball names and layout directly verified via GitHub API + archive inspection, but the "install correctly on both arches" end-to-end flow is untested on real hardware
- Architecture: MEDIUM — the `afterPack` signing pattern is derived from documented hook semantics and confirmed sign-scope gaps, but not yet exercised against this specific codebase's staged output
- Pitfalls: MEDIUM-HIGH — AMFI/Gatekeeper distinction and the layout-guardrail gap are both grounded in direct source reads of this repo's own files, not speculation

**Research date:** 2026-09-17
**Valid until:** 30 days (stable domain — electron-builder major version and python-build-standalone release cadence are the main things that could shift this)

## RESEARCH COMPLETE
