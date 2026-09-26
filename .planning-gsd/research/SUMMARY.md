# Project Research Summary

**Project:** orgtree
**Domain:** Porting a Windows-only Electron + Python desktop app to a native, unsigned, local macOS build
**Researched:** 2026-09-17
**Confidence:** MEDIUM

## Executive Summary

Orgtree is an Electron + Python-engine desktop app that currently only ships on Windows (NSIS installer, Windows Scheduled Task autostart, `ctypes`/`taskkill` process management, taskbar-specific APIs). This milestone is a macOS port (MAC-01: unsigned, local build, no Apple Developer account) rather than a new product — every research thread confirms this is a well-trodden "port an Electron app to macOS" problem with direct, known equivalents for each Windows-only mechanism, not a novel architecture problem.

The recommended approach is additive, not a rewrite: keep `electron-builder` (already pinned at 26.15.3) and add a `mac` target instead of adopting a second packaging tool; bundle a `python-build-standalone` relocatable interpreter as the macOS analog of the existing Windows embeddable-Python pattern; replace the Windows Scheduled Task with a `launchd` LaunchAgent; and swap Windows-only APIs (`ctypes.windll`, `taskkill`, Windows taskbar/AppUserModelId calls) for their direct POSIX/Electron-macOS equivalents (`os.kill`/`psutil`, `os.killpg` on a process group, `app.dock.bounce()`). Notably, the codebase's `gitrunner.py` and `liveness.py` already contain correct POSIX branches for process lifecycle — these are the reference patterns to copy, not something to design from scratch.

The dominant risk is not architectural but platform-enforcement: Apple Silicon's AMFI kernel-level code-signing check will refuse to launch a fully unsigned `.app` even with zero network/download involvement, producing a misleading "app is damaged" dialog. Ad-hoc signing (`mac.identity: "-"`) is a one-line, zero-cost mitigation that should be adopted even though full notarization is explicitly out of scope for this milestone. Secondary risks are packaging details (`.icns` required, not `.ico`) and reliability caveats on Electron APIs that silently degrade when unsigned (`openAtLogin` may not take effect without a signed/notarized build — treat as best-effort, not guaranteed).

## Key Findings

### Recommended Stack

No new packaging tool is needed. `electron-builder` already builds the Windows NSIS target; add a `mac` target to the same config. Bundle CPython via `python-build-standalone` (astral-sh), the same "stage a runtime, then pip-install into it" pattern `tools/provision-runtime.py` already implements for Windows, just pointed at the `aarch64-apple-darwin`/`x86_64-apple-darwin` `install_only` tarballs. Autostart uses `launchd` (OS-native, no dependency) via a LaunchAgent plist, loaded with the modern `launchctl bootstrap gui/<uid> <plist>` (not the deprecated `load`/`unload`).

**Core technologies:**
- electron-builder 26.15.3 (existing, pinned): produces `.app`/`.dmg`/`.zip` — reuse the existing Windows config, add a `mac` target, no version bump needed.
- python-build-standalone (astral-sh, `3.10.x`, `install_only` tarball): relocatable CPython bundled inside the `.app` — direct macOS analog of the existing Windows embeddable-Python zip; same de facto standard used by `uv`/Briefcase/PyOxidizer.
- launchd (macOS built-in, no dependency): boot/login-time autostart of the Python engine — direct replacement for the Windows Scheduled Task, using `launchctl bootstrap`/`bootout` (not the legacy, deprecated `load`/`unload`).

**Packaging notes carried forward:** use `mac.identity: "-"` (ad-hoc signing) rather than `null` — zero cost, removes the manual "unsigned developer" approval prompt; skip `entitlements.mac.plist`/`hardenedRuntime` entirely (irrelevant, and a source of failure modes, when unsigned); generate a `.icns` via `iconutil` — `.ico` is silently ignored or errors on the mac target; notarization (`@electron/notarize`, Apple Developer creds) is explicitly out of scope for MAC-01 and should stay unconfigured.

### Expected Features

Feature research focused on macOS-native parity for the four Windows-only integrations already in the codebase: taskbar AppUserModelId grouping, `flashFrame`-based attention flashing, electron-updater's Squirrel.Windows auto-update, and Scheduled-Task boot autostart. None have a 1:1 Windows API, but each has a well-understood macOS swap-in.

**Must have (table stakes):**
- Dock "needs attention" bounce (`app.dock.bounce('critical')`) — direct swap for `taskbar-attention.ts`'s Windows flash logic; works fine unsigned, no signing dependency.
- `.icns` app icon and a proper `mac` electron-builder target — without these the app looks/feels like an unported Windows shim.
- launchd-based boot autostart of the Python engine — parity with the existing Windows Scheduled Task behavior.

**Should have (competitive/convenience):**
- "Launch Orgtree at login" Electron toggle via `app.setLoginItemSettings({ openAtLogin: true })` — but flagged MEDIUM reliability: Electron's own docs warn this goes through `SMAppService` and may silently no-op on an unsigned/unnotarized build. Ship as best-effort, verify by hand, don't treat as guaranteed.

**Defer (v2+):**
- Full code signing + notarization (Developer ID cert, `hardenedRuntime`, `entitlements.mac.plist`, `@electron/notarize` with `afterSign` hook) — only needed if distributing beyond the developer's own machine; explicitly out of scope for MAC-01.
- Squirrel.Mac-based auto-update — requires signing to function correctly; revisit once a signing identity exists.

### Architecture Approach

The port is a mechanical swap of three Windows-only subsystems for their POSIX/macOS equivalents, all handled in the existing Python engine layer (`engine/backend/orgtree/`) with no new dependencies for the core mechanism (`psutil` is already an approved dependency and should be reused where richer process state is needed, not re-justified).

**Major components:**
1. **PID liveness probing** — replace `ctypes.windll.kernel32.OpenProcess`/`GetExitCodeProcess` with `os.kill(pid, 0)` or `psutil.pid_exists`/`psutil.Process`. `liveness.py::_observe_pid` already has a correct POSIX branch (lines 190-202) — verify under macOS CI, don't rewrite.
2. **Process-group kill** — replace `taskkill /PID N /T /F` with spawning into its own process group (`start_new_session=True`) and `os.killpg(pgid, signal.SIGKILL)`. `gitrunner.py::_stop`/`run()` already implements this exact pattern correctly (lines 37-43, 72) — copy it verbatim as the reference for any other Windows-only kill logic.
3. **CLI tool resolution** — replace hardcoded `.exe`/`.cmd` invocation via `cmd.exe /c` with `shutil.which("name")` and direct exec, no shell wrapper.

### Critical Pitfalls

1. **Apple Silicon (arm64) AMFI enforcement blocks unsigned binaries even offline** — a locally-built `.app` with zero quarantine attribute still fails to launch on arm64 Macs with a misleading "app is damaged" dialog, because AMFI is a kernel-level check independent of Gatekeeper/quarantine. Mitigate with ad-hoc signing (`mac.identity: "-"`) at build time — zero cost, no certificate required.
2. **`.ico`-only app icon breaks/silently degrades mac packaging** — electron-builder's mac target requires `.icns`; generate one via `iconutil` from an `.iconset` and point `build.mac.icon` at it, leaving the existing `.ico` untouched for the Windows target.
3. **`openAtLogin` (SMAppService) may silently no-op without signing/notarization** — Electron's own docs flag this; treat as best-effort for an unsigned build, verify manually rather than assuming success.
4. **Deprecated `launchctl load`/`unload`** — still functional but deprecated since macOS 10.10 with known per-user/per-session domain edge cases; use `launchctl bootstrap gui/<uid> <plist>` / `bootout` instead, and validate the plist with `plutil -lint` before install.
5. **Nested unsigned binaries inside the `.app` bundle** — Electron apps ship dozens of loose `.dylib`/`.node`/framework binaries; a partial signing pass can leave some nested binaries unsigned even if the top-level bundle appears signed, re-triggering the AMFI failure from pitfall #1. Ensure ad-hoc signing covers the full bundle, not just the top-level `.app`.

## Implications for Roadmap

Based on research, suggested phase structure:

### Phase 1: macOS packaging foundation
**Rationale:** Everything else (autostart, process management, feature parity) depends on having a `.app` that actually launches on the target hardware first. This is also where the highest-severity pitfall (AMFI/arm64 signing) lives — get it wrong here and every later phase is untestable on a real Mac.
**Delivers:** electron-builder `mac` target producing a launchable, ad-hoc-signed `.app`/`.dmg` with a correct `.icns` icon; bundled `python-build-standalone` runtime staged the same way the Windows embeddable-Python zip is staged today.
**Addresses:** Table-stakes packaging (icon, launchability) from FEATURES.md.
**Avoids:** Pitfall #1 (AMFI signing), Pitfall #2 (`.icns` requirement), Pitfall #5 (nested unsigned binaries).

### Phase 2: Process lifecycle port (POSIX subprocess management)
**Rationale:** The engine's process management is Windows-only in three specific spots; this is pure mechanical porting with reference implementations already in the codebase (`gitrunner.py`, `liveness.py`), so it's low-risk and should happen right after packaging works, before autostart/UI-integration phases depend on a working engine process.
**Delivers:** `os.kill`/`psutil`-based liveness checks and `os.killpg`-based process-group termination wired into any remaining Windows-only code paths, following the existing `gitrunner.py`/`liveness.py` POSIX branches as the copy-paste reference.
**Uses:** stdlib `os`, `signal`, `subprocess`, `shutil.which`, plus the already-approved `psutil` dependency.
**Implements:** Architecture component 1 (PID liveness) and 2 (process-group kill) from ARCHITECTURE.md.

### Phase 3: launchd autostart
**Rationale:** Depends on Phase 1's packaged `.app` existing at a stable path and Phase 2's process management being POSIX-correct, since the LaunchAgent is what invokes the Python engine at boot/login.
**Delivers:** A generated LaunchAgent plist (validated with `plutil -lint`), installed/removed via `launchctl bootstrap`/`bootout`, replacing the Windows Scheduled Task.
**Addresses:** Table-stakes boot autostart parity from FEATURES.md.
**Avoids:** Pitfall #4 (deprecated `launchctl load`/`unload`).

### Phase 4: macOS UI/OS-integration parity
**Rationale:** Cosmetic/UX parity (dock attention, login-item toggle) is the least architecturally risky and most independently testable slice — sequence it last so it doesn't block the harder packaging/process work, and so any `openAtLogin` reliability caveat is discovered against an already-working signed build.
**Delivers:** `app.dock.bounce('critical')` replacing `taskbar-attention.ts`'s flash logic; best-effort `app.setLoginItemSettings({ openAtLogin: true })` toggle with manual verification noted in the UI.
**Addresses:** Should-have feature (login toggle) and remaining table-stakes item (dock attention) from FEATURES.md.
**Avoids:** Pitfall #3 (`openAtLogin` silent no-op on unsigned builds — document as best-effort, don't over-promise in UI copy).

### Phase Ordering Rationale
- Packaging must come first: nothing else can be verified on real hardware until the `.app` launches at all (AMFI signing is the single highest-severity blocker).
- Process-lifecycle porting comes second because it's low-risk (reference patterns already exist in-repo) and is a dependency for the engine that autostart will invoke.
- Autostart depends on both a stable packaged path and correct process management, so it comes third.
- UI/OS-integration parity is the most isolated, least risky work and is deferred last so any reliability caveats (login item) are tested against a build that's already known to launch and sign correctly.

### Research Flags

Phases likely needing deeper research during planning:
- **Phase 1 (packaging):** MEDIUM confidence only — electron-builder config was verified against primary docs, but python-build-standalone and launchd specifics were verified against secondary/community sources, not primary Apple docs. Flag for `--research-phase` if signing/entitlements edge cases surface.

Phases with standard patterns (skip research-phase):
- **Phase 2 (process lifecycle):** HIGH confidence — verified directly against existing correct POSIX code in the codebase (`gitrunner.py`, `liveness.py`); this is a copy-the-existing-pattern task, not new research.
- **Phase 3 (launchd) and Phase 4 (UI parity):** Well-documented Electron/Apple APIs with clear official-doc-backed swap-ins; standard patterns.

## Confidence Assessment

| Area | Confidence | Notes |
|------|------------|-------|
| Stack | MEDIUM | electron-builder config verified via context7 against official docs; python-build-standalone and launchd findings verified against community/third-party docs, not primary Apple/astral-sh pages |
| Features | MEDIUM | Electron/electron-builder docs via Context7 = MEDIUM tier; Squirrel.Mac signing requirement cross-checked across multiple GitHub issues + Electron code-signing docs |
| Architecture | HIGH | Verified directly against Python stdlib docs AND existing correct POSIX branches already present in the codebase (`gitrunner.py`, `liveness.py`) |
| Pitfalls | MEDIUM | Grounded in Apple/Electron official docs cross-referenced with developer-forum explanations; WebSearch was non-functional this session, narrowing source breadth |

**Overall confidence:** MEDIUM

### Gaps to Address
- python-build-standalone and launchd findings rest on secondary sources, not primary astral-sh/Apple documentation — validate against the actual `install_only` tarball behavior and a real LaunchAgent install during Phase 1/3 execution, not just at planning time.
- `openAtLogin` reliability under `SMAppService` for an unsigned build is a documented caveat, not a tested outcome — verify manually on real Apple Silicon hardware during Phase 4 rather than assuming the API call succeeds.
- WebSearch was non-functional during pitfalls research, narrowing source breadth for that file — if new pitfalls surface during Phase 1 (signing) execution, treat as expected and budget time for them rather than assuming full coverage.

## Sources

### Primary (HIGH confidence)
- context7 (electron-builder library docs) — mac target config, identity/signing options
- Python stdlib docs (`os`, `signal`, `subprocess`, `shutil`) — POSIX process management APIs
- Existing codebase (`engine/backend/orgtree/gitrunner.py`, `liveness.py`) — verified-correct existing POSIX branches

### Secondary (MEDIUM confidence)
- Electron official docs (Context7) — `app.dock.bounce`, `setLoginItemSettings`/`SMAppService` caveat, code-signing docs
- electron/electron GitHub issues #8204, #36640; electron-builder #3983 — Squirrel.Mac signing requirement cross-reference
- Apple developer-forum explanations — AMFI/arm64 signing enforcement behavior

### Tertiary (LOW confidence)
- Community docs for python-build-standalone (astral-sh) and launchd `bootstrap`/`bootout` usage — not verified against primary astral-sh/Apple pages; recommend spot-checking during Phase 1/3

---
*Research completed: 2026-09-17*
*Ready for roadmap: yes*
