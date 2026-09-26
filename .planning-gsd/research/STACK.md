# Stack Research

**Domain:** macOS packaging for an existing Windows-only Electron + Python desktop app (electron-builder mac target, portable Python runtime, launchd autostart)
**Researched:** 2026-09-17
**Confidence:** MEDIUM (electron-builder config verified against official/source docs via context7; Python-runtime and launchd findings verified against community/third-party docs, not the primary source pages — see Gaps)

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| electron-builder | 26.15.3 (already pinned in `package.json`) | Produces the macOS `.app`/`.dmg`/`.zip` artifacts | Already the packaging tool for the Windows build (NSIS target) — adding a `mac` target to the same config keeps one build tool instead of introducing a second (e.g. Electron Forge). No version bump needed for this milestone. |
| python-build-standalone (astral-sh org, formerly indygreg/gregoryszorc) | latest `3.10.x` build tag available for the target arch | Relocatable, self-contained CPython interpreter to bundle inside the `.app` | Direct macOS analog of the Windows embeddable-Python zip the project already uses in `tools/provision-runtime.py`. Ships architecture-specific `install_only` tarballs (`aarch64-apple-darwin`, `x86_64-apple-darwin`) that run from any directory with no system Python install — same "stage a runtime, then pip-install deps into it" pattern the Windows path already implements, so `provision-runtime.py`/`stage-runtime.mjs` gain a macOS branch instead of a rewrite. It's the de facto standard: `uv`, Briefcase, and PyOxidizer all consume the same artifacts. |
| launchd (LaunchAgent, no new dependency — OS-native) | macOS built-in | Boot/login-time autostart of the Python engine process | Direct replacement for the Windows Scheduled Task + registry Run-key combo (MAC-08). A per-user LaunchAgent plist under `~/Library/LaunchAgents/` needs no admin/root privilege and no code signing — matching the "unsigned, local build" scope of this milestone. `launchd` is the only supported mechanism on macOS; there is no equivalent to `schtasks`/registry autostart. |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| `plist` (npm) | latest 3.x | Serialize a JS object into the LaunchAgent `.plist` XML (and read it back for status/uninstall logic) | Use instead of hand-templating XML strings — avoids manual escaping bugs in `ProgramArguments`/paths, and is the same library electron-builder itself uses internally for `Info.plist` generation, so no new trust surface is introduced. |
| `iconutil` (macOS built-in CLI, not an npm package) | ships with Xcode CLT | Convert a `.iconset` folder of PNGs into the `.icns` electron-builder needs for `build.mac.icon` | Since the mac build must run on a Mac anyway (electron-builder cannot cross-build a signed/relocatable mac target from Windows), `iconutil` is already present — no need for an npm icon-conversion package (`electron-icon-builder`, `png2icons`) unless CI later needs to generate icons from a non-mac runner. |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| `plutil -lint` | Validate LaunchAgent plist syntax before install | Run against the generated plist in CI/dev to catch malformed XML before `launchctl bootstrap` fails silently. |
| `launchctl bootstrap gui/<uid> <plist>` / `launchctl bootout gui/<uid> <plist>` | Load/unload the LaunchAgent | Use these (macOS 10.10+) rather than the deprecated `launchctl load`/`unload` — `load`/`unload` still work but are legacy and print deprecation warnings. |
| `codesign -dv --verbose=4 <app>` | Inspect signing state during dev | Confirms whether electron-builder actually left the app unsigned vs. ad-hoc signed (see Stack Patterns below) — useful when debugging "app won't open" on a teammate's Apple Silicon Mac. |

## Installation

```bash
# No new core npm dependency required for electron-builder mac target —
# electron-builder 26.15.3 already supports mac/dmg/zip targets, just unused in package.json today.

# Supporting library for LaunchAgent plist generation
npm install plist

# Python runtime: not an npm/pip install — download the release asset directly
# in tools/provision-runtime.py's new macOS branch, e.g.:
#   https://github.com/astral-sh/python-build-standalone/releases/download/<tag>/cpython-<ver>+<tag>-<arch>-apple-darwin-install_only.tar.gz
# where <arch> is aarch64 or x86_64 depending on the build host / target.
```

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|--------------------------|
| python-build-standalone `install_only` tarball | PyInstaller / Nuitka one-file freeze of the engine | Only if the project later wants to ship the engine as a single compiled binary instead of an interpreter + site-packages tree. This would be a structural rewrite of `tools/provision-runtime.py`/`stage-runtime.mjs` (they currently stage an interpreter and pip-install into it, mirroring the Windows embeddable-zip flow) — not recommended for this milestone since it breaks platform parity with the working Windows approach for no immediate benefit. |
| Per-user LaunchAgent (`~/Library/LaunchAgents`) | System-wide LaunchDaemon (`/Library/LaunchDaemons`) | Only if the engine must run before any user logs in or must run as root/for all users. Requires `sudo` at install time and is a much closer parity break from a per-user Windows Scheduled Task — avoid unless a concrete requirement emerges. |
| Hand-rolled LaunchAgent plist via `plist` npm package | Electron's `app.setLoginItemSettings()` | `setLoginItemSettings()` only auto-launches the Electron *app* itself at login (mac equivalent of a Windows registry Run key) — it cannot register a separate background process (the Python engine) to launch independently of the app opening. Since MAC-08 is specifically about autostarting the *engine*, this API doesn't cover the requirement and a real LaunchAgent is still needed. |
| `mac.identity: null` (skip signing) for this milestone | `mac.identity: "-"` (ad-hoc signing) | Ad-hoc signing avoids the one-time "unsigned developer" approval prompt in System Settings on Apple Silicon and is a one-line config change (`identity: "-"`) — worth adopting even for local builds since it removes a manual approval step for every fresh build, at zero cost (no certificate needed). |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|--------------|
| `@electron/notarize` / notarization config (`afterSign` hook, `APPLE_ID`/`APPLE_API_KEY` env vars) | Out of scope for MAC-01 ("unsigned, local build") — notarization requires an Apple Developer account, a Developer ID certificate, and network calls to Apple's notary service. It's already present as a transitive dependency of electron-builder but is never invoked unless the notarization env vars are set. | Leave unconfigured. Revisit only when/if the project needs to distribute the build outside the developer's own machine (a later milestone, not this one). |
| `entitlements.mac.plist` / `hardenedRuntime: true` | Hardened runtime and entitlements only matter for signed+notarized distribution builds; they have no effect (and add failure modes, e.g. missing `com.apple.security.cs.allow-jit` breaking Electron's V8 JIT) when `mac.identity` is `null` or `"-"`. | Skip entirely for this milestone; electron-builder does not require an entitlements file when signing is skipped. |
| `.ico` as the mac app icon | electron-builder's mac target requires `.icns` specifically — `.ico` is Windows-only and will be silently ignored or error depending on version. | Generate a `.icns` via `iconutil` from a `.iconset` (see Supporting Libraries) and point `build.mac.icon` at it; keep the existing `.ico` for the Windows target unchanged. |
| `launchctl load` / `launchctl unload` (legacy subcommands) | Deprecated since macOS 10.10 in favor of `bootstrap`/`bootout`; still functional today but prints deprecation warnings and has known edge cases with the per-user vs. per-session domain distinction. | Use `launchctl bootstrap gui/<uid> <plist>` and `launchctl bootout gui/<uid> <plist>`. |

## Stack Patterns by Variant

**If the dev/build machine is Apple Silicon (arm64) — the common case today:**
- Target `mac.target: [{ target: "dmg", arch: ["arm64"] }, { target: "zip", arch: ["arm64"] }]` and bundle the `aarch64-apple-darwin` python-build-standalone tarball.
- Because this milestone's scope is a local unsigned build, don't build `universal` (combining x86_64+arm64 via `lipo`) yet — it doubles the Python runtime bundle size and build time for no requirement in MAC-01..MAC-10. Revisit only if the app needs to run on both Apple Silicon and Intel Macs.

**If a teammate/CI machine is Intel (x86_64):**
- Swap to `arch: ["x64"]` and the `x86_64-apple-darwin` python-build-standalone tarball. `tools/provision-runtime.py`'s new macOS branch should detect `platform.machine()` and pick the matching asset automatically, same as it presumably already branches on Windows arch today.

**If the app later needs real distribution (post this milestone):**
- Add a Developer ID certificate, switch `mac.identity` to the real identity string, add `entitlements.mac.plist` with `com.apple.security.cs.allow-jit` + `com.apple.security.cs.allow-unsigned-executable-memory` (both required for Electron's V8 under hardened runtime), enable `hardenedRuntime: true`, and configure `afterSign` with `@electron/notarize` using `APPLE_API_KEY`/`APPLE_API_KEY_ID`/`APPLE_API_ISSUER`. None of this is needed now.

## Version Compatibility

| Package A | Compatible With | Notes |
|-----------|------------------|-------|
| electron-builder 26.15.3 | `mac.identity` (flat option) | Correct syntax for this pinned version. electron-builder v27 moved macOS signing options under a new `mac.sign` object (`mac.sign.identity` replaces `mac.identity`) — do not follow v27-era docs/examples verbatim until/unless the project bumps electron-builder past v27. |
| python-build-standalone CPython build | engine's current Python 3.10.13 target | Pick the closest available `3.10.x` release tag from the astral-sh/python-build-standalone releases page — tags are date-stamped (`<python-version>+<YYYYMMDD>`) and change frequently, so pin the exact tag string once chosen and verify at implementation time (see Sources below) rather than trusting a version number written today. |
| Electron 44.2.0 | macOS 11+ (Big Sur or later) | Recent Electron majors (44.x) require a fairly modern macOS baseline; if the target machine runs an older macOS this should be confirmed against Electron's own platform support table before packaging. |

## Sources

- `/electron-userland/electron-builder` (Context7 library ID) — mac target config, `mac.identity`/`mac.sign.identity`, ad-hoc signing glossary entry, v27 breaking-changes migration table, DMG icon/background config, entitlements.mac.plist example. Confidence: MEDIUM (official repo docs via Context7, not independently cross-verified).
- https://www.launchd.info/ — LaunchAgent plist keys (Label, ProgramArguments, RunAtLoad, KeepAlive, etc.), LaunchAgents vs LaunchDaemons scope/location, `launchctl bootstrap`/`bootout` vs legacy `load`/`unload`. Confidence: LOW (third-party reference site, not Apple's own docs — cross-check against `man launchd.plist` before implementation).
- https://github.com/astral-sh/python-build-standalone — org/license (astral-sh, MPL-2.0) and repo identity confirmed directly; per-arch macOS build naming (`aarch64-apple-darwin`/`x86_64-apple-darwin`, `install_only` tarballs) confirmed via general web search, not the repo's own README/docs pages (those fetches returned stale/empty content). Confidence: LOW — verify the exact current release tag and asset names against the live releases page before wiring up `provision-runtime.py`.
- WebSearch (general web results) — supplementary confirmation of the unsigned-build config pattern. Confidence: LOW.

---
*Stack research for: macOS Electron + Python packaging port*
*Researched: 2026-09-17*
