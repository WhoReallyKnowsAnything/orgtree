# Domain Pitfalls

**Domain:** Porting a Windows-only Electron + Python-engine desktop app to macOS (unsigned local build, launchd autostart, Python sidecar process)
**Researched:** 2026-09-17
**Confidence:** MEDIUM (grounded in Apple/Electron official docs + cross-referenced developer-forum explanations; WebSearch tool was non-functional this session so breadth is narrower than ideal — see Gaps note in Sources)

## Critical Pitfalls

Mistakes that cause rewrites, silent feature loss, or multi-day debugging on a machine the team doesn't have easy iteration access to.

### 1. Apple Silicon (arm64) refuses to execute an improperly-signed app — even with zero internet involvement
**What goes wrong:** A locally-built, completely unsigned `.app` that was never downloaded (no `com.apple.quarantine` attribute at all) still fails to launch on an Apple Silicon Mac with the generic, misleading dialog "App is damaged and can't be opened. You should move it to the Trash."
**Why it happens:** On Intel Macs, code-signature checks are gated by the quarantine flag (Gatekeeper). On Apple Silicon, AMFI (AppleMobileFileIntegrity) enforcement is a *kernel-level* requirement, independent of Gatekeeper/quarantine: every Mach-O executable must carry at least a valid signature (ad-hoc is fine) to be allowed to run at all. Electron apps ship dozens of loose `.dylib`/`.node`/framework binaries; a naive or partial signing pass (or none at all) leaves some of those nested binaries unsigned or invalidly signed, and the whole bundle is rejected.
**Consequences:** The v1 build target is explicitly "unsigned, local build" (MAC-01/MAC-09) — this pitfall will hit on the very first launch attempt on the team's actual (almost certainly Apple Silicon) dev Mac, and the error message actively misdirects debugging toward "corrupted download" rather than "needs a signature."
**Prevention:** Every packaged build must run a deep ad-hoc signing pass after packaging: `codesign --force --deep -s - YourApp.app`, then verify with `codesign --verify --deep --strict --verbose=2 YourApp.app`. Better: configure electron-builder to ad-hoc sign nested binaries correctly during packaging rather than a manual post-hoc `--deep` pass, since `--deep` can miss entitlement requirements Electron's JIT/multi-process helpers need. Bake this signing step into the `package:mac` script (MAC-01) so it can never be skipped.
**Detection:** `codesign -dv --verbose=4 YourApp.app` on every nested `Contents/Frameworks/*.framework` and helper binary; add a build-smoke-test step that actually launches the packaged `.app` (not just `npm start`) before calling MAC-09 done.

### 2. Gatekeeper quarantine still applies whenever the build leaves the machine that built it
**What goes wrong:** The moment the packaged `.app` is zipped, AirDropped, uploaded/downloaded, or copied via a network share, macOS attaches `com.apple.quarantine`, and first launch shows "'YourApp' cannot be opened because it is from an unidentified developer" (a different dialog from Pitfall 1, with a different fix).
**Why it happens:** Any file-transfer path that goes through LaunchServices/Safari/Mail/AirDrop sets the quarantine xattr; a build that only worked because it was tested directly on the build machine will fail for a teammate who receives a zipped copy.
**Consequences:** Confuses "it worked when I built it" with "it's fine to distribute" — teams routinely discover this only when someone other than the builder tries the app.
**Prevention:** Document the two required unblocks for anyone who receives a copy: right-click → Open (accept once), or `xattr -cr YourApp.app` before first launch. If any semi-automated distribution (internal file share, GitHub Release asset) is used even informally, note this prominently in a README/CONTRIBUTING note — do not assume "local build" means "this never comes up."
**Detection:** `xattr -l YourApp.app` shows `com.apple.quarantine` when present; test the actual handoff path (zip → transfer → unzip → open) at least once, not just repeated builds on one machine.

### 3. Sidecar Python binaries lose their executable bit and/or their own valid signature during packaging
**What goes wrong:** The bundled Python runtime (interpreter binary, `.so` native-extension files) either loses `+x` permission during packaging/git/zip round-trips, or — on arm64 — fails AMFI/Gatekeeper checks as an independently-executed Mach-O binary even though the parent `.app` launches fine.
**Why it happens:** `extraResources` copies preserve most metadata but permission bits and code signatures are easy to lose in a packaging pipeline that wasn't designed with a nested-executable sidecar in mind (this is exactly the gap flagged in `tools/provision-runtime.py` / MAC-03 — there is currently no macOS runtime provisioning at all, so this hasn't been exercised yet). Pitfall 1's per-binary signing requirement applies to the sidecar interpreter and every native `.so` extension inside it, not just the top-level `.app`.
**Consequences:** The engine process (the actual product value — agent orchestration) fails to spawn with an `EACCES` or a silent "damaged"-style rejection, while the Electron shell itself launches fine — this looks like an engine bug, not a packaging bug, and can burn significant debugging time misdirected at the wrong layer.
**Prevention:** After provisioning the mac Python runtime (MAC-03, via `python-build-standalone` mac builds per CONCERNS.md item 9), explicitly `chmod +x` the interpreter and any bundled native binaries as a packaging step, then include them in the same deep-signing pass as Pitfall 1. Verify the *specific* spawn path the engine uses (`engine.ts` reading `options.python`) actually executes post-package, not just pre-package in the dev tree.
**Detection:** Smoke-test MUST spawn the packaged (not dev-tree) Python interpreter and run one real agent job end-to-end (MAC-09's own acceptance bar) — a dev-tree-only test will not catch this.

### 4. launchd's failure model is fundamentally different from Windows Task Scheduler and can permanently disable the autostart without the app knowing
**What goes wrong:** Windows Scheduled Task (S4U) retries per its own configured history quietly in the background. macOS's LaunchAgent equivalent (MAC-08) does three things Windows does not: (a) shows a system notification the first time it registers — "[App] Background Items Added", deep-linking to System Settings → General → Login Items & Extensions; (b) if the job crashes repeatedly in a short window, launchd's throttling escalates to macOS **automatically flipping the Login Items toggle off** and surfacing "[App] was disabled because it kept quitting unexpectedly"; (c) there is **no programmatic API for the app to re-enable itself** after that — only the user, manually, in System Settings.
**Why it happens:** This is deliberate Apple UX policy (crash-loop protection + user transparency for background items), not a bug — but it means the boot-engine's crash-restart behavior (currently modeled on Windows Scheduled Task semantics per CONCERNS.md item 5) cannot be a 1:1 port. A flaky early build of the engine that crash-loops during development will get its own autostart silently disabled by macOS, and the app has no way to detect or recover from that state except asking the user to go flip a switch.
**Consequences:** This is exactly the kind of thing that's cheap to design for up front and expensive to retrofit — if MAC-08 is built assuming "launchd = Task Scheduler with different file syntax," the team will not learn about the auto-disable behavior until a real crash loop happens, likely during dogfooding, and it will look like "autostart randomly stopped working."
**Prevention:** Set `ThrottleInterval` deliberately in the LaunchAgent plist to avoid rapid-fire restart storms; on app startup, have the desktop app check `SMAppService` status (`.enabled` / `.requiresApproval` / `.notRegistered`) and surface an in-app banner ("background engine is disabled — click to re-enable in System Settings") rather than silently failing to attach. Use `SMAppService.openSystemSettingsLoginItems()` to deep-link the user there directly.
**Detection:** Deliberately crash the engine 10+ times in quick succession during MAC-08 testing and confirm the app detects and surfaces the disabled state, rather than just confirming "it starts at login" once.

### 5. No macOS equivalent exists yet for Windows' Job Object process-tree containment — orphaned agent processes will leak
**What goes wrong:** The Windows build's "guardian" model uses a Job Object to guarantee that killing the engine kills its entire descendant tree. The current POSIX fallback (already present per CONCERNS.md items 6 and 11) is a bare `process.kill(pid, 'SIGKILL')` on the single known PID — if any spawned provider CLI (Claude Code, Codex, Antigravity, OpenRouter processes) forks grandchildren, macOS will leak those as orphans on every crash, restart, or forced-kill.
**Why it happens:** Job Objects have no POSIX analogue; the correct replacement (process groups) was never wired up because it wasn't needed on Windows.
**Consequences:** Silent, cumulative resource leakage — CPU/memory creep that isn't visible until days into real usage, and by the time it's noticed ("why is my Mac's fans spinning"), the leaked processes are disconnected from any UI, hard to attribute back to the app, and hard to reproduce on demand.
**Prevention:** Spawn engine/provider-CLI children with `detached: true` in their own session on darwin, and kill via the negative PID (process group): `process.kill(-pid, 'SIGKILL')`, matching the fix approach CONCERNS.md item 6 already recommends. Confirm this specifically against every provider CLI in scope (Claude Code, Codex, Antigravity, OpenRouter) since each may have a different subprocess-forking depth.
**Detection:** `ps -ef` / Activity Monitor process count before and after several kill-and-restart cycles of each provider type; a growing baseline process count after the engine reports "stopped" is the signature of this bug.

### 6. TCC treats every rebuild of an unsigned/ad-hoc app as a "different app," causing permission-prompt whack-a-mole during development
**What goes wrong:** Any permission the app requests (camera/mic/full-disk/local-network/accessibility, even for legitimate later features) can silently re-prompt or silently deny on every single rebuild during development, and developers interpret this as "permissions are broken" rather than "the code signature changed."
**Why it happens:** macOS's TCC database keys permission grants to the app's code signature/identity. An unsigned or ad-hoc-signed app gets a new effective identity on every rebuild (no stable Developer ID), so TCC cannot reliably recognize "this is still the same app I already approved." Separately, any permission request without the matching `Info.plist` usage-description key (`NSCameraUsageDescription`, `NSLocalNetworkUsageDescription`, etc.) fails **silently** — no crash, no error, the feature just doesn't work — which is easy to mistake for an application bug.
**Consequences:** Hours lost to "why did this permission stop working" during iteration, and features that appear broken but are actually just denied by TCC with no visible error.
**Prevention:** Add every usage-description key the app will ever need up front in electron-builder's `mac.extendInfo` block, even before the corresponding feature ships. When permissions misbehave during development, `tccutil reset <Service> <bundle-id>` (or reset entirely) before assuming a code bug. Verify actual entitlements post-build with `codesign -d --entitlements :- YourApp.app` rather than trusting the source config was applied.
**Detection:** If a permission worked yesterday and silently doesn't today with no code change to the feature itself, suspect a signature-identity change from the latest rebuild before debugging the feature logic.

### 7. Local Network privacy gate can silently break engine↔desktop communication if binding ever drifts from strict loopback
**What goes wrong:** macOS Sonoma+ gates any LAN device discovery or connection to private-range/multicast addresses behind a "[App] would like to find and connect to devices on your local network" prompt — and if `NSLocalNetworkUsageDescription` isn't in `Info.plist`, the connection is silently denied rather than erroring clearly.
**Why it happens:** The engine's HTTP+bearer-token channel is documented as loopback-only, and pure `127.0.0.1` direct socket connections are *not* gated by this prompt — so this is a low-probability pitfall today, but any future change that binds to `0.0.0.0`, uses mDNS/Bonjour for engine discovery, or listens on a LAN-visible interface would trip it invisibly.
**Consequences:** If it ever does trip, the failure mode is "connection silently fails" rather than a clear permission-denied error, which is expensive to diagnose without knowing this gate exists.
**Prevention:** Keep the engine bind strictly to `127.0.0.1` (verify this explicitly, don't just assume it — CONCERNS.md doesn't call this out one way or the other) and add `NSLocalNetworkUsageDescription` proactively anyway, since it costs nothing and future-proofs against an inadvertent bind-address change.
**Detection:** `lsof -iTCP -sTCP:LISTEN` on the engine process to confirm it's bound to `127.0.0.1` and not `*`/`0.0.0.0`.

### 8. Zero macOS test coverage means every pitfall above will surface on a real Mac, not in CI
**What goes wrong:** CONCERNS.md confirms there are currently no macOS-specific tests and no macOS CI runner/matrix entry at all — every item above (signing, launchd, sidecar spawn, TCC) currently ships with zero regression coverage on the target platform.
**Why it happens:** The existing Windows-only test suite (installer/elevation/taskbar probes) has no mac equivalents yet, and several Windows-only tests don't even self-guard with a platform check (CONCERNS.md item 13), so a naive "just run the suite on mac" will produce false failures unrelated to the real macOS-specific work.
**Consequences:** Every pitfall in this document will be discovered manually, late, and expensively unless tests are added alongside each fix rather than after.
**Prevention:** Add a macOS smoke test that packages the app, runs the deep-signing pass, launches the packaged `.app` (not `npm start`), and completes one real agent job end-to-end — this single test would catch Pitfalls 1, 3, and partially 5. Add platform guards to the untested Windows-only `.ps1`-backed tests before they're ever run on a mac CI runner.
**Detection:** N/A — this pitfall's entire nature is the absence of detection; treat MAC-10 as blocking, not optional cleanup.

## Moderate Pitfalls

### 1. GUI-launched apps do not inherit a login shell's PATH
**What goes wrong:** Provider CLIs (`claude`, `codex`, etc.) installed via Homebrew/npm-global/nvm work fine when the engine is run from a Terminal during development, then fail to spawn ("command not found") once the app is double-clicked from Finder or launched via `launchd`/Login Items.
**Prevention:** Resolve provider CLI paths explicitly (absolute path lookup, e.g. via `/usr/libexec/path_helper` or a controlled `$PATH` extension covering `/opt/homebrew/bin`, `~/.local/bin`, common nvm shim paths) rather than relying on `spawn()`'s inherited environment. This is exactly the gap CONCERNS.md item 12 already flags as needing real testing on the POSIX branch of `providerlogin.ts`.

### 2. `flashFrame()` silently no-ops on macOS
**What goes wrong:** The Windows "taskbar pulse" attention feature (`taskbar-attention.ts`) has no platform gate and will run on mac without error, but does nothing — a product-decision feature quietly absent with no test failure to flag it.
**Prevention:** Branch to `app.dock.bounce('informational')` / `app.dock.cancelBounce(id)` on darwin (CONCERNS.md item 7's own fix approach).

### 3. Tray icons render oversized/wrong-colored without macOS-specific assets
**What goes wrong:** `.ico`-only tray assets are not template images; macOS menu-bar `Tray` icons need black+alpha `Template.png` + `@2x` variants or they render incorrectly.
**Prevention:** Add macOS-specific tray asset set and switch selection by platform (CONCERNS.md item 8).

### 4. Windows-shaped Python runtime layout assumptions may silently resolve to the wrong interpreter
**What goes wrong:** `stage-runtime.mjs`/`runtime-layout.mjs` validate a `Lib/site-packages` + `._pth`-file layout that is CPython-Windows-specific; a naive port can pass its own layout assertions while the app silently falls back to whatever `python3` is on the system PATH instead of the bundled runtime.
**Prevention:** Adapt layout assertions to the POSIX `lib/pythonX.Y/site-packages` shape (CONCERNS.md item 9), and explicitly assert in a test that the *bundled* interpreter's path — not just any working interpreter — is what gets launched.

## Minor Pitfalls

### 1. Unguarded `ctypes.windll.kernel32` calls fail loudly, not subtly, on import
**What goes wrong:** `liveness.py` / `antigravityrun.py` will raise `AttributeError`/`ImportError` immediately at import time on macOS if left unguarded — this is a hard, obvious failure rather than a silent bug, but only if it's actually exercised by a macOS test run (see Critical Pitfall 8).
**Prevention:** `os.name`/`sys.platform` branch with a macOS-safe equivalent or explicit no-op per CONCERNS.md's MAC-05.

### 2. `.ico`-only app icon fails or silently defaults during mac packaging
**What goes wrong:** electron-builder requires a `.icns` for the macOS app icon; without one, packaging either errors or falls back to a generic Electron icon depending on version.
**Prevention:** Generate `.icns` alongside the existing `.ico` (MAC-02) and verify it's referenced in the `mac` build block, not just present on disk.

### 3. Hardcoded `C:\`/`SystemRoot` test fixtures are test-only risk
**What goes wrong:** `containment.test.ts`/`eventdecode.test.ts` reference literal Windows paths; low production impact but will produce confusing failures if run unguarded on mac CI (CONCERNS.md item 14).
**Prevention:** Low priority — confirm these are test-only and don't accidentally assert Windows-only redaction regexes before deprioritizing further.

## Phase-Specific Warnings

| Phase Topic | Likely Pitfall | Mitigation |
|-------------|----------------|-------------|
| Packaging config / mac target (MAC-01, MAC-02) | Build succeeds but produces an app that macOS refuses to execute (Critical #1) | Bake a deep ad-hoc codesign + verify step into `package:mac`; don't treat "electron-builder exits 0" as done |
| Python runtime provisioning (MAC-03) | Sidecar interpreter/native extensions lose exec bit or fail per-binary signing (Critical #3) | chmod +x + include sidecar binaries in the same signing pass; smoke-test the packaged (not dev-tree) interpreter |
| POSIX branches for exec resolution (MAC-04, MAC-12) | GUI-launched app doesn't see the same PATH as a Terminal (Moderate #1) | Resolve provider CLI paths explicitly, test by launching from Finder, not just `npm start` |
| ctypes/Windows API calls (MAC-05) | Loud but easy-to-miss import-time failures if untested on mac (Minor #1) | Cover with a real macOS test run, not just code review |
| Process containment (MAC-06) | Orphaned provider-CLI descendant processes leak on every kill/crash (Critical #5) | Process-group spawn + negative-PID kill on darwin; stress-test with repeated crash/restart cycles |
| Taskbar/dock APIs (MAC-07) | Silent no-op feature loss (Moderate #2) | Explicit darwin branch to dock bounce, verified visually |
| launchd autostart (MAC-08) | launchd's crash-loop auto-disable has no app-side recovery API, unlike Windows Task Scheduler (Critical #4) | Design for `SMAppService` status checks + in-app re-enable prompt from day one, not as an afterthought |
| End-to-end mac build/launch/agent-job (MAC-09) | Everything above compounds here — this is where Critical #1, #3, #5 all surface simultaneously if not caught earlier | Treat this milestone's acceptance test as the canonical macOS smoke test, and run it on real hardware, not CI alone until MAC-10 lands |
| Test coverage (MAC-10) | Zero current macOS coverage means all of the above ships blind (Critical #8) | Add tests alongside each fix, not after; audit existing `.ps1`-backed tests for missing platform guards before enabling any mac CI matrix entry |

## Sources

- Apple Developer documentation — notarization/Gatekeeper overview (`developer.apple.com/documentation/security/notarizing-macos-software-before-distribution`) — HIGH-tier source, but retrieved content was thin on macOS-specific Gatekeeper enforcement mechanics; supplemented by cross-referenced explanation below.
- Apple Support — Gatekeeper user-facing behavior (`support.apple.com/en-us/102445`) — official Apple source, LOW confidence per this session's tooling classification (single webfetch, unverified) despite being a primary source.
- Apple Developer Forums thread on Apple Silicon AMFI signature enforcement — cross-referenced explanation of why unsigned/ad-hoc-signed apps fail on arm64 independent of quarantine; the single most load-bearing finding in this document (Critical Pitfall 1). Treat as MEDIUM-confidence pending a direct spot-check on the team's actual dev Mac.
- Apple `SMAppService` documentation (`developer.apple.com/documentation/servicemanagement/smappservice`) — official source on Login Items banner/notification behavior, crash-loop auto-disable, and the lack of a programmatic re-enable API (Critical Pitfall 4).
- launchd.info community reference — KeepAlive/RunAtLoad/ThrottleInterval semantics.
- Electron official docs — code signing (`electronjs.org/docs/latest/tutorial/code-signing`) — safeStorage/login-item/autoUpdater code-signing dependencies.
- Cross-referenced developer discussion on camera-permission entitlements failure in production electron-builder apps — TCC/entitlements/usage-description pitfalls (Critical Pitfall 6).
- `.planning/codebase/CONCERNS.md` (this repo, committed `ea1eeea`) — primary source of truth for the 14 known Windows-only code paths; every pitfall above was cross-checked against it to avoid duplicating what's already tracked as a portability blocker versus what's a *new* macOS-security-specific risk this research adds.

**Gaps / tooling note:** The built-in `WebSearch` tool returned a persistent API error for the entire duration of this research session (all queries failed with "text content blocks must be non-empty"), and the `websearch` provider fallback (Brave) had no API key configured. All findings above come from targeted `WebFetch` calls to specific authoritative URLs plus domain synthesis, not broad web search — coverage of edge cases (e.g., specific electron-builder version regressions, less common provider-CLI spawning quirks) is narrower than a full research pass would achieve. Recommend a follow-up pass with a working search tool before finalizing MAC-08/MAC-09 phase plans, and a direct hands-on spot-check of Critical Pitfalls 1 and 4 on the team's actual Apple Silicon Mac as early as possible — both are cheap to verify and expensive to discover mid-phase.
