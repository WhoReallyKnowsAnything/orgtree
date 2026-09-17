# Phase 3: Launchd Autostart - Research

**Researched:** 2026-09-17
**Domain:** macOS `launchd`/`launchctl` per-user LaunchAgent lifecycle (install/uninstall/detect), replacing a Windows Scheduled Task
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

#### Plist install/uninstall mechanism
- **D-01:** Use a per-user LaunchAgent plist written to `~/Library/LaunchAgents/`.
- **D-02:** Install/uninstall via the modern `launchctl bootstrap gui/<uid> <plist>` /
  `launchctl bootout gui/<uid> <plist>` verbs — not the deprecated `launchctl load`/`unload`.
- **D-03:** Validate the generated plist with `plutil -lint` before install, mirroring the
  `Prepare`/`Register`/`Remove`/`Stop` action shape of the existing Windows
  `Invoke-BootLifecycle` contract (`InstallDir`, `InstallMode` params carry over conceptually;
  `OperatorSid` has no macOS equivalent — per-user LaunchAgents are inherently user-scoped).

#### Crash-loop detection
- **D-04:** macOS silently disables a LaunchAgent that crash-loops (repeated fast-exit) with
  no programmatic re-enable available. The app must detect this condition and prompt the user
  rather than failing to start with no explanation (BOOT-02). This is a first-class task, not
  an afterthought — it needs its own detection check (e.g. on next manual launch, check
  whether the LaunchAgent is loaded/running via `launchctl print gui/<uid>/<label>` and whether
  it was expected to have started this session) and a user-facing prompt/remediation path.
  **This research corrects the mechanism behind D-04's claim — see Summary and Pattern 3 below.
  D-04's outcome requirement (detect + prompt, never fail silently) stands; the OS-level
  explanation it was based on does not survive verification as originally stated.**

#### Verification approach
- **D-05:** Research flags `launchctl bootstrap`/`bootout` usage as resting on secondary
  community sources, not primary Apple documentation. The plan must include a real verification
  step (actually install/uninstall a test LaunchAgent and observe `launchctl print` / process
  state) rather than trusting the research claims as-is. **This research performs that real
  verification step directly (see Pattern 2 and Sources) — the plan should still schedule the
  longer-duration crash-loop observation called out in Open Question 3, which this session's
  ~90-second test could not cover.**

### Claude's Discretion
- Exact plist key set (`Label`, `ProgramArguments`, `RunAtLoad`, `KeepAlive` shape, log paths)
  and exact crash-loop detection heuristic (which `launchctl print` fields / exit-code pattern
  to key off) are open for the planner/implementer to determine, as long as D-01 through D-05
  hold.

### Deferred Ideas (OUT OF SCOPE)
- `setLoginItemSettings`-based GUI toggle for autostart (DIST-03) — explicitly deferred to a
  later phase per REQUIREMENTS.md, layered on top of this phase's LaunchAgent.
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research support |
|----|-------------|-------------------|
| BOOT-01 | Engine autostart on login is implemented via a per-user `launchd` LaunchAgent plist (`~/Library/LaunchAgents/`), replacing the Windows Scheduled Task + registry mechanism | Standard Stack, Pattern 1 (plist template + `plutil -lint` gate, verified keys), Pattern 2 (verified live `bootstrap`/`bootout` cycle), Pitfalls 2/3/6, Validation Architecture test map |
| BOOT-02 | App detects and surfaces to the user when macOS has silently disabled the LaunchAgent after a crash loop (no programmatic re-enable exists — must prompt, not fail silently) | Pattern 3 (corrected detection design: `launchd` throttle vs. `launchctl`-level disable vs. BTM disable), Pitfall 1/4, Assumptions A2/A3, Open Question 3 |
</phase_requirements>

## Summary

This phase replaces `tools/register-boot-engine.ps1` / `unregister-boot-engine.ps1` / `boot-engine-task.ps1` (a SYSTEM-scoped, S4U, all-users-installer-owned Scheduled Task) with a per-user `launchd` LaunchAgent written to `~/Library/LaunchAgents/`, installed/removed with the modern `launchctl bootstrap`/`bootout` verbs, and validated with `plutil -lint`. Everything the user's discuss-phase context flagged as needing primary-source verification (D-05) has now been directly confirmed: either read from the actual `man launchd.plist(5)` / `man launchctl(1)` pages shipped on this Darwin 27.2 (arm64) machine, or reproduced live by bootstrapping, observing, and booting out a real crash-looping test LaunchAgent on this machine today.

**The single most important correction this research makes to the existing CONTEXT.md decisions:** D-04 states "macOS silently disables a LaunchAgent that crash-loops... with no programmatic re-enable available." A live empirical test (a LaunchAgent with `KeepAlive=true` that exits 1 immediately) shows this is **not** what actually happens at the `launchd` level — `launchd` throttles the respawn to the `ThrottleInterval` default of 10 seconds and retries **forever**; it never set a "disabled" bit and never appeared in `launchctl print-disabled` after 10 observed crash cycles over 75 seconds. The mechanism that actually matches "silently disabled, no programmatic re-enable" is a **separate, undocumented subsystem**: macOS's Background Task Management (BTM), queryable via the private `sfltool dumpbtm` tool, which tracks every legacy LaunchAgent/LaunchDaemon with a `Disposition` (`enabled/disabled`, `allowed/disallowed`, `notified/not-notified`) surfaced in System Settings → Login Items & Extensions → "Allow in the Background." A user (or, per widely-reported Ventura-era bugs, the OS itself in edge cases) can toggle an item to `disallowed`, and there is no `launchctl`-level command to undo that — the only sanctioned path back is the System Settings toggle. BOOT-02's detection logic should be designed around this corrected mechanism, not around an imagined `launchd`-level auto-disable.

**Primary recommendation:** Generate the plist as a small, fixed XML template (5-6 keys — do not add a `plist` npm dependency for this), gate every install behind `plutil -lint`, install/remove with `launchctl bootstrap gui/<uid> <path>` / `launchctl bootout gui/<uid>/<label>`, set `KeepAlive.SuccessfulExit = false` (not bare `KeepAlive = true`) to match the engine's existing exit-code contract, and detect trouble on next manual app launch by checking (a) whether `launchctl print gui/<uid>/<label>` finds the service at all, and (b) whether the label appears in `launchctl print-disabled gui/<uid>`, treating both `launchctl print`'s output and `sfltool dumpbtm` as unstable/private diagnostics rather than an API.

## Architectural Responsibility Map

| Capability | Primary Tier | Secondary Tier | Rationale |
|------------|-------------|----------------|-----------|
| LaunchAgent plist generation + install/uninstall | Frontend Server (Electron main process) | — | Unsigned local `.app`/`.dmg` builds have no installer-script hook (no NSIS `installer.nsh` equivalent for `dmg`/`zip` targets — `package.json` build config has no `mac` postinstall step yet). Unlike Windows, where `build/installer.nsh` owns task registration and "the running APPLICATION never registers, elevates or stops the task itself" `[VERIFIED: docs/engine-contract.md:15]`, macOS has no installer to own this — the app itself must register/unregister at runtime. |
| Engine autostart execution at login | OS Service (`launchd`, outside the app's process tree) | API/Backend (Python `engine/service_host.py` invoked as the job) | Once bootstrapped, `launchd` owns spawning independently of whether the Electron app is running. |
| Crash-loop / disabled-state detection | Frontend Server (Electron main process) | — | `launchd` has no push/webhook to notify the app; detection must poll `launchctl print` / `launchctl print-disabled` on next manual launch. |
| User-facing remediation prompt | Browser/Client (renderer UI) | Frontend Server (IPC bridge) | Dialog renders in the renderer, triggered via IPC from the main-process detection check, mirroring the existing `handle(...)`/IPC pattern in `apps/desktop/main/index.ts`. |

## Standard Stack

### Core

No new dependency is required. `launchctl` and `plutil` are Darwin OS-native command-line tools, present on every macOS 10.10+ install with zero package footprint. Both were confirmed present and functional on this machine's Darwin 27.2 (arm64) build via live invocation (see Verification below).

| Tool | Source | Purpose | Why standard |
|------|--------|---------|--------------|
| `launchctl` | macOS system binary (`/bin/launchctl`) | Bootstrap/bootout/print/enable/disable LaunchAgents | The only supported interface to `launchd`; no library wraps it because there is nothing to wrap — it is a thin CLI over Mach IPC. `[VERIFIED: man launchctl(1), local]` |
| `plutil` | macOS system binary (`/usr/bin/plutil`) | Validate plist syntax before install (`-lint`) | Ships with the OS; exits 0/1 predictably, scriptable. `[VERIFIED: man plutil(1) + live test, local]` |

### Supporting

| Library | Version | Purpose | When to use |
|---------|---------|---------|-------------|
| Node built-in `child_process` (`execFileSync`/`spawnSync`) | bundled with Electron 44.2.0's Node runtime | Invoke `launchctl`/`plutil` from the Electron main process | Already the codebase's convention for shelling out (see `apps/desktop/main/providerlogin.ts`'s `resolveArgv`/spawn patterns `[VERIFIED: apps/desktop/main/providerlogin.ts:82]`) — no new dependency needed. |

### Alternatives Considered

| Instead of | Could use | Tradeoff |
|------------|-----------|----------|
| Hand-rolled plist XML template + `plutil -lint` gate | An npm `plist` library to build/serialize the XML | Rejected: the schema is small and fixed (Label, ProgramArguments, RunAtLoad, KeepAlive, StandardOutPath/StandardErrorPath, WorkingDirectory — at most 7 keys, all scalar/array, no nested user data beyond trusted internal paths). The existing Windows equivalent (`tools/boot-engine-task.ps1`'s `New-BootTaskXml`) already hand-builds XML via `[xml]` DOM manipulation and then validates the result — the project's own precedent is "build simple XML, validate it," not "add a dependency for a 6-key file." Adding a library here is the kind of unrequested "flexibility" the project's own engineering norms warn against. |
| `launchctl bootstrap`/`bootout` | `launchctl load`/`unload` | Rejected per D-02 and confirmed deprecated: `man launchctl(1)` does not list `load`/`unload` under primary subcommands in the same section as `bootstrap`/`bootout`/`enable`/`disable`/`print` — they appear later (line 358) as a legacy compatibility path. `bootstrap`/`bootout` is what Apple's own current documentation and the man page structure treat as canonical. |
| Electron `app.setLoginItemSettings({ openAtLogin: true })` | Raw `launchctl bootstrap` on a hand-written plist | Explicitly out of scope (DIST-03, deferred). Also: per prior project research, this Electron API routes through `SMAppService` on modern macOS and is flagged unreliable on unsigned/unnotarized builds — not a substitute for BOOT-01's direct-plist approach. `[CITED: prior project research/SUMMARY.md:39]` |

**Installation:** None — no `npm install` / `pip install` needed for this phase.

## Package Legitimacy Audit

**Not applicable.** This phase installs zero external packages (no npm, no PyPI). It uses only OS-native `launchctl`/`plutil` binaries and the Node `child_process` module already bundled with Electron. Skip the Package Legitimacy Gate entirely — there is nothing to audit.

## Architecture Patterns

### System Architecture Diagram

```
┌─────────────────────────────┐
│  Electron main process       │  (apps/desktop/main/*.ts)
│                               │
│  1. On darwin, first run OR  │
│     explicit "enable auto-   │
│     start" action:           │
│     ┌───────────────────┐    │
│     │ generate plist     │    │  writes ~/Library/LaunchAgents/
│     │ (template string)  │───┼─▶ com.maurdekye.orgtree.boot-engine.plist
│     └─────────┬───────────┘    │
│               ▼                │
│     ┌───────────────────┐      │
│     │ plutil -lint       │──── │  exit 0 = proceed, exit 1 = abort+report
│     └─────────┬───────────┘    │
│               ▼                │
│     ┌───────────────────┐      │
│     │ launchctl bootstrap│──── │──▶  launchd (OS)
│     │ gui/<uid> <path>   │      │        │
│     └───────────────────┘      │        │ RunAtLoad / KeepAlive
│                                 │        ▼
│  2. On every manual app launch:│   engine/service_host.py
│     ┌───────────────────┐      │   (python-build-standalone runtime,
│     │ launchctl print     │◀───┼───   path confirmed by Phase 1/2)
│     │ gui/<uid>/<label>   │      │
│     └─────────┬───────────┘      │
│               ▼                  │
│     ┌───────────────────┐        │
│     │ not found, or in    │       │
│     │ print-disabled?     │───────┼──▶ IPC → renderer: remediation dialog
│     └───────────────────┘        │      ("Login Items & Extensions" deep link)
│                                   │
│  3. On uninstall/disable action: │
│     launchctl bootout             │
│     gui/<uid>/<label>  ───────────┼──▶ launchd removes job; rm plist file
└─────────────────────────────┘
```

### Recommended Project Structure

```
apps/desktop/main/
├── launchagent-mac.ts     # NEW: generate plist, lint, bootstrap/bootout, detect (darwin-only)
├── index.ts                # existing: call launchagent-mac functions on darwin at appropriate lifecycle points
tests/
├── disruptive/
│   └── launchagent-install.test.mjs   # NEW: real bootstrap/bootout side effects (matches existing tests/disruptive/ convention)
└── launchagent-detect.test.mjs        # NEW: pure detection-logic unit tests against mocked launchctl output
```

### Pattern 1: Plist template + lint gate (mirrors existing PS1 precedent)

**What:** Build the plist as a template string, write it to disk, run `plutil -lint` on the written file, and only proceed to `bootstrap` if lint exits 0.
**When to use:** Every install, always — this is D-03's explicit requirement.
**Example (verified plist keys and their documented semantics — `man launchd.plist(5)`, local):**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.maurdekye.orgtree.boot-engine</string>
    <key>ProgramArguments</key>
    <array>
        <string>/path/to/engine/runtime/bin/python3</string>
        <string>/path/to/engine/service_host.py</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <dict>
        <key>SuccessfulExit</key>
        <false/>
    </dict>
    <key>WorkingDirectory</key>
    <string>/path/to/engine</string>
    <key>StandardOutPath</key>
    <string>/path/to/logs/boot-engine.out.log</string>
    <key>StandardErrorPath</key>
    <string>/path/to/logs/boot-engine.err.log</string>
</dict>
</plist>
```
`Label` uses the app's own reverse-DNS `appId` (`com.maurdekye.orgtree` `[VERIFIED: package.json:34]`) as a namespace prefix — this is the Apple-documented convention ("it is the expected convention for launchd property list files to be named `<Label>.plist`" `[VERIFIED: man launchd.plist(5), local]`) and avoids collisions with the Windows task name `"Orgtree Background Engine"` `[VERIFIED: tools/boot-engine-task.ps1:3, quoting `$script:BootTaskName = 'Orgtree Background Engine'`]`, which has no macOS equivalent.

**`KeepAlive.SuccessfulExit = false` is not a guess** — it is required to match the engine's own documented shutdown contract: *"Requested shutdown exits 0 so restart-on-failure settings do not resurrect a deliberately stopped engine; engine failure exits nonzero."* `[VERIFIED: docs/engine-contract.md:16]`. Plain `KeepAlive = true` would respawn the engine even after the user (or app) deliberately stops it with exit 0 — `SuccessfulExit: false` restarts the job only on a non-successful (nonzero) exit, which is exactly this contract.

**The `ProgramArguments` first element is `[ASSUMED]`** — `engine/runtime/bin/python3` is the conventional `python-build-standalone` `install_only` tarball layout (`python/bin/python3` after extraction) `[ASSUMED — community/GitHub sources, not astral-sh's own docs]`, by analogy with the verified Windows equivalent `path.join(directory, 'runtime', 'python.exe')` `[VERIFIED: apps/desktop/main/index.ts:1275]`. **Neither the exact runtime subpath nor the interpreter filename can be verified in this session** — Phase 1 (packaging/runtime) and Phase 2 (process lifecycle) have not yet produced any artifacts in this worktree (`.planning/phases/` contains only `03-launchd-autostart/`). The planner must treat this path as a placeholder to be confirmed against Phase 1/2's actual output before or during Phase 3 execution — this is explicitly called out in CONTEXT.md's "Reusable Assets" section as unresolved.

### Pattern 2: Install/remove lifecycle (verified live, this session)

**What:** `launchctl bootstrap`/`bootout` both accept either a domain-target alone, a domain-target plus one or more plist paths, or a domain-target plus a `service-target` (domain + label) — confirmed by `man launchctl(1)`: `"bootstrap | bootout domain-target [service-path service-path2 ...] | service-target"` `[VERIFIED: man launchctl(1), local, line 73]`.
**When to use:** Install always passes the plist path; uninstall may pass either the path or `gui/<uid>/<label>`.
**Verified live on this machine (Darwin 27.2, arm64), full cycle:**
```bash
# Install (path form) — exit 0, no output on success
launchctl bootstrap gui/501 "/path/to/com.maurdekye.orgtree.boot-engine.plist"

# Observe (best-effort diagnostic only — see Pitfall 3)
launchctl print gui/501/com.maurdekye.orgtree.boot-engine
#   state = spawn scheduled
#   runs = 1
#   last exit code = 1
#   spawn type = daemon (3)

# Uninstall (service-target form) — exit 0
launchctl bootout gui/501/com.maurdekye.orgtree.boot-engine

# Post-removal: print reports the job is gone
launchctl print gui/501/com.maurdekye.orgtree.boot-engine
#   Bad request.
#   Could not find service "com.maurdekye.orgtree.boot-engine" in domain for user gui: 501
```
This exact sequence — plist written to a scratch path (not even `~/Library/LaunchAgents`), `plutil -lint` → OK, `bootstrap` → exit 0, `print` showing live state, `bootout` → exit 0, `print` afterward failing cleanly — was run end-to-end on this machine during this research session. `[VERIFIED: empirical test, this session, 2026-09-17]`

### Pattern 3: Crash-loop / disabled-state detection (corrected from D-04)

**What:** Two independent, separately-queryable states must not be conflated:

1. **`launchd`-level throttling (not a failure state to report):** A `KeepAlive`d job that keeps exiting nonzero is retried forever at the `ThrottleInterval` cadence (10s default `[VERIFIED: man launchd.plist(5), local: "by default, jobs will not be spawned more than once every 10 seconds"]`). This is normal `launchd` behavior, not a "disabled" condition — do not surface this as "autostart is broken," since the job **is** still trying.
2. **`launchd`-level persistent disable (`launchctl enable`/`disable`):** `"Once a service is disabled, it cannot be loaded in the specified domain until it is once again enabled. This state persists across boots of the device."` `[VERIFIED: man launchctl(1), local, line 90-93]`. Queryable via `launchctl print-disabled gui/<uid>` — `"Prints the list of disabled services in the specified domain."` `[VERIFIED: man launchctl(1), local, line 229]`. **This is programmatically re-enableable** (`launchctl enable gui/<uid>/<label>`), contradicting D-04's "no programmatic re-enable" framing for this specific mechanism.
3. **Background Task Management (BTM) disable — the actual "no programmatic re-enable" case:** A separate, undocumented-by-Apple subsystem tracks every legacy LaunchAgent/LaunchDaemon and surfaces it in System Settings → Login Items & Extensions → "Allow in the Background," queryable (on this machine) via the private tool `sfltool dumpbtm`, which showed live per-user entries with a `Disposition` field, e.g. `Disposition: [enabled, allowed, notified]` for a real user LaunchAgent at `/Users/501/Library/LaunchAgents/...` `[VERIFIED: empirical test, `sfltool dumpbtm`, this session]`. When a user (or the OS, per widely-reported Ventura-era "Background Items" bugs) flips an item to `disallowed`, there is genuinely no `launchctl` command to undo it — the only path back is the System Settings toggle. **This is the real match for D-04's "silently disabled, no programmatic re-enable" framing**, not `launchd`'s own crash-loop response, which this research empirically found does *not* auto-disable.

**Recommended detection logic** (best-effort, layered):
```typescript
// apps/desktop/main/launchagent-mac.ts (sketch — not verified against real Phase 1/2 paths)
function checkAutostartHealth(uid: number, label: string): 'ok' | 'not-installed' | 'disabled' | 'unknown' {
  const printResult = spawnSync('launchctl', ['print', `gui/${uid}/${label}`])
  if (printResult.status !== 0) return 'not-installed'          // service-target not found — primary, stable signal (exit code, not text)

  const disabledList = spawnSync('launchctl', ['print-disabled', `gui/${uid}`])
  if (disabledList.stdout?.toString().includes(label)) return 'disabled'   // launchd-level disable (re-enableable)

  // sfltool dumpbtm is undocumented/private — treat as enrichment only, never as the sole signal.
  // A parse failure here must fall through to 'ok', not throw.
  return 'ok'
}
```
Prefer **exit codes** over parsing `launchctl print`'s stdout text wherever possible — the man page is explicit that this output *"is NOT API in any sense at all... may change from release to release without warning"* `[VERIFIED: man launchctl(1), local, line 222-224]`.

### Anti-Patterns to Avoid

- **Bootstrapping from a scratch/temp path and calling it "installed":** `launchctl bootstrap` accepts any path and will run the job for the current `launchd` session regardless of where the file lives — this was directly verified (`/private/tmp/.../com.orgtree.research.crashtest.plist` bootstrapped and ran fine). But `launchd` only auto-rescans `~/Library/LaunchAgents` (and the other paths listed in `man launchd.plist(5)`'s FILES section) **at next login** `[VERIFIED: man launchd.plist(5), local, FILES section]`. The plist **must** be physically copied into `~/Library/LaunchAgents/<Label>.plist` (D-01) for autostart to survive a reboot — bootstrapping from elsewhere only starts it for the current session.
- **Treating `launchctl print` field names as a stable contract:** see Pattern 3 — the man page disclaims this explicitly.
- **Relying on `sfltool dumpbtm` as a primary signal:** it is not in any `man` page, has no public Apple documentation, and its output format could change without notice. Use it only as best-effort diagnostic enrichment for a remediation message, never as the sole gate for "is autostart broken."

## Don't Hand-Roll

| Problem | Don't build | Use instead | Why |
|---------|-------------|-------------|-----|
| Detecting whether the plist file is well-formed | A hand-rolled XML validator | `plutil -lint` (D-03) | It is the OS-shipped, purpose-built tool; exits 0/1 predictably (`[VERIFIED: live test, this session — valid plist → exit 0, "<file>: OK"; truncated plist → exit 1, "Encountered unexpected EOF"]`). |
| Tracking "how many times has this crashed" | A custom in-app crash counter duplicating `launchd`'s own accounting | `launchd`'s built-in `ThrottleInterval` + `runs`/`last exit code` fields (best-effort) | `launchd` already throttles and counts; duplicating this logic in the app risks disagreeing with the OS's own behavior and doubles the maintenance surface. |
| Re-implementing "is this a login item" UI | A custom autostart-status settings screen from scratch | Deep-link the user to System Settings → Login Items & Extensions for BTM-level remediation (the `disallowed` case has no programmatic fix anyway) | Apple's own UI is the only place a BTM `disallowed` disposition can actually be reversed — building an in-app equivalent cannot bypass this. |

**Key insight:** Almost everything this phase needs is a thin, well-documented OS binary interface (`launchctl`, `plutil`) with a genuinely small, fixed-shape config file. The temptation to add abstraction (a plist library, a custom crash-loop state machine, a custom login-items UI) should be resisted — the project's own Windows-side precedent (hand-rolled XML + validation, no external dependency) and this phase's tiny scope both point the same direction.

## Common Pitfalls

### Pitfall 1: Confusing `launchd`'s throttle with a "disabled" state
**What goes wrong:** Treating every crash-loop as "autostart is broken, alert the user" when `launchd` is actually retrying fine, just slowly.
**Why it happens:** D-04's original framing (pre-verification) assumed crash-looping itself triggers a silent OS-level disable.
**How to avoid:** Only surface a remediation prompt for the two states that are genuinely terminal: not-found (`launchctl print` exit ≠ 0) or `launchctl print-disabled` / BTM `disallowed`. A live throttled retry loop is not, by itself, one of these.
**Warning signs:** Users report "the app never starts at login" but `launchctl print` shows `state = spawn scheduled` with a climbing `runs` counter — that is a crash in the engine itself (a Phase 1/2 concern), not an autostart-mechanism failure.

### Pitfall 2: Plist bootstrapped from a non-canonical path won't survive reboot
**What goes wrong:** Install code bootstraps directly from wherever the app's resources live, without copying into `~/Library/LaunchAgents/`; autostart works immediately (current session) but silently stops working after the next full reboot.
**Why it happens:** `bootstrap` doesn't require the file to live in `~/Library/LaunchAgents` to work *right now* — verified directly on this machine from a `/private/tmp` path.
**How to avoid:** Always write the final plist to `~/Library/LaunchAgents/<Label>.plist` (D-01) and bootstrap from that exact path, not a resources/temp copy.
**Warning signs:** Install/uninstall tests pass in CI (single session) but a manual "reboot and check" test (D-05's mandated real verification step) fails.

### Pitfall 3: `KeepAlive: true` fights the engine's own clean-shutdown contract
**What goes wrong:** User quits the app / stops the engine deliberately; `launchd` immediately respawns it because bare `KeepAlive: true` restarts on any exit, including a clean `exit 0`.
**Why it happens:** `KeepAlive: true` doesn't examine the exit code at all — verified in `man launchd.plist(5)`: `"The value may be set to true to unconditionally keep the job alive."`
**How to avoid:** Use `KeepAlive: { SuccessfulExit: false }`, matching the documented contract that requested shutdown exits 0 (don't restart) and failure exits nonzero (do restart) `[VERIFIED: docs/engine-contract.md:16]`.
**Warning signs:** The engine process becomes effectively unkillable from the UI — every deliberate stop is immediately undone by `launchd`.

### Pitfall 4: Parsing `launchctl print` output as if it were stable
**What goes wrong:** A regex keyed on exact field text (`"state = spawn scheduled"`) breaks silently on a future macOS release when Apple reformats the output.
**Why it happens:** The man page explicitly disclaims stability, but it's easy to miss.
**How to avoid:** Prefer exit-code checks (found/not-found) as the primary signal; treat any text-field parsing as best-effort, always with a safe fallback to "unknown" rather than a crash or a false "disabled" report.
**Warning signs:** Detection logic throws or mis-reports after an unrelated macOS point release.

### Pitfall 5: Guessing the packaged Python interpreter path
**What goes wrong:** `ProgramArguments`'s first element is hardcoded to a guessed `runtime/bin/python3` path that doesn't match what Phase 1's `python-build-standalone` staging actually produces.
**Why it happens:** Phase 1/2 have not executed in this worktree yet — there is no ground truth to read.
**How to avoid:** Before finalizing the plist template, confirm the exact runtime path against Phase 1's actual packaging output (or `apps/desktop/main/index.ts`'s darwin branch, once Phase 1/2 land) rather than trusting this research's `[ASSUMED]` placeholder.
**Warning signs:** `launchctl print` shows the service bootstrapped fine but `last exit code` is always the shell's "command not found" code (127) or similar.

### Pitfall 6: Deprecated `load`/`unload` mixed with `bootstrap`/`bootout`
**What goes wrong:** A stray `launchctl load` (e.g. from a stale test, a developer's manual debugging, or copy-pasted older documentation) loads the job into the legacy per-session namespace; a subsequent `launchctl bootstrap` of the "same" job can then fail with an "already loaded"-style I/O error because the two subsystems track overlapping state inconsistently.
**Why it happens:** `load`/`unload` still exist (documented later in `man launchctl(1)`, line 358) and plenty of older tutorials/StackOverflow answers use them.
**How to avoid:** Use only `bootstrap`/`bootout`/`print`/`print-disabled`/`enable`/`disable` anywhere in this phase's code and its tests — never `load`/`unload` (D-02).
**Warning signs:** Intermittent "already loaded" or "Input/output error" failures that don't reproduce on a clean bootstrap-only sequence.

## Code Examples

### Bootstrap install, verified live this session
```bash
plutil -lint /path/to/com.maurdekye.orgtree.boot-engine.plist   # exit 0 required before proceeding
launchctl bootstrap gui/$(id -u) /path/to/com.maurdekye.orgtree.boot-engine.plist
```
Source: `man plutil(1)` + `man launchctl(1)`, both read locally on this machine; sequence executed live, this session.

### Bootout uninstall, verified live this session
```bash
launchctl bootout gui/$(id -u)/com.maurdekye.orgtree.boot-engine
rm -f ~/Library/LaunchAgents/com.maurdekye.orgtree.boot-engine.plist
```

### Health check (exit-code-first, best-effort text enrichment)
```bash
if ! launchctl print "gui/$(id -u)/com.maurdekye.orgtree.boot-engine" >/dev/null 2>&1; then
  echo "not-installed-or-not-bootstrapped"
elif launchctl print-disabled "gui/$(id -u)" | grep -q "com.maurdekye.orgtree.boot-engine"; then
  echo "disabled (launchctl enable gui/$(id -u)/com.maurdekye.orgtree.boot-engine to fix)"
else
  echo "ok (best-effort — verify with a manual reboot per D-05)"
fi
```

## State of the Art

| Old approach | Current approach | When changed | Impact |
|--------------|------------------|---------------|--------|
| `launchctl load`/`unload` | `launchctl bootstrap`/`bootout` | Documented as the modern interface since OS X 10.10 (Yosemite) era `launchd` rewrite | `load`/`unload` still function (present in `man launchctl(1)`) but operate on a different, legacy per-session model; mixing the two causes inconsistent state (Pitfall 6). |

**Deprecated/outdated:**
- `launchctl load -w` / `unload -w`: superseded by `bootstrap`/`bootout` + `enable`/`disable` for persistence control — still present in the man page but documented later, alongside other legacy subcommands.

## Assumptions Log

| # | Claim | Section | Risk if wrong |
|---|-------|---------|----------------|
| A1 | The packaged macOS Python interpreter lives at `engine/runtime/bin/python3` (python-build-standalone `install_only` layout) | Pattern 1 (Code Example), Pitfall 5 | `ProgramArguments` in the generated plist points at a nonexistent binary; the LaunchAgent bootstraps successfully but the job fails immediately with a POSIX exec error, which then reads (misleadingly) like a crash-loop rather than a path bug. Must be confirmed against Phase 1's actual packaging output before this phase's plist template is finalized. |
| A2 | `sfltool dumpbtm`'s `Disposition` field semantics (`enabled/disabled`, `allowed/disallowed`) map cleanly onto "silently disabled, no programmatic re-enable" for the purposes of BOOT-02 | Pattern 3 | `sfltool` is undocumented by Apple; its exact behavior around crash-triggered (vs. only user-triggered) disposition changes was not, and likely cannot be, fully reproduced in a ~90-second research session. A longer real-world crash-loop observation (hours, not seconds) is the D-05-mandated verification the plan must still schedule. |
| A3 | No automatic `launchd`-level "disable" is ever triggered purely by repeated fast-exit crashes, at any timescale | Summary, Pattern 3 | Only ~10 crash cycles over ~75 seconds were observed live in this session. It remains possible (though nothing in `man launchd.plist(5)`/`man launchctl(1)` documents it, and none appeared in `launchctl print-disabled` during the test) that a much longer or more aggressive crash pattern behaves differently. Treat as high-confidence but not exhaustively proven; the plan's D-05 verification step should include a longer-duration observation if schedule allows. |

## Open Questions

1. **Exact Phase 1/2 packaged Python interpreter and engine entrypoint paths**
   - What we know: Windows precedent is `path.join(directory, 'runtime', 'python.exe')` for the interpreter and `engine/service_host.py` for the boot-specific entrypoint `[VERIFIED: apps/desktop/main/index.ts:1275, docs/engine-contract.md:15]`.
   - What's unclear: The exact macOS subpath `python-build-standalone`'s `install_only` layout will produce once Phase 1 actually stages it, and whether `engine/service_host.py` is reused unchanged or needs a macOS-specific headless launcher.
   - Recommendation: The planner should add an explicit task to confirm these paths against Phase 1/2's real output (or coordinate directly with whichever phase lands first) before finalizing the plist template — do not ship the `[ASSUMED]` path from this research without that confirmation.

2. **Where in the app lifecycle should install/uninstall actually be triggered?**
   - What we know: Windows registration is entirely installer-owned (`build/installer.nsh`); the running app "never registers, elevates or stops the task itself" `[VERIFIED: docs/engine-contract.md:15]`. macOS has no installer-script equivalent for an unsigned local `.app`/`.dmg`/`.zip` build.
   - What's unclear: Whether install should happen automatically on first launch, behind an explicit user-facing toggle/setting, or both — CONTEXT.md's "Claude's Discretion" doesn't address the trigger point, only the plist keys and detection heuristic.
   - Recommendation: Default to first-launch automatic install (matching "autostart works without the user launching it manually," success criterion 1) with an uninstall path wired to app uninstall/removal if one exists, or at minimum a manual "disable autostart" action — this is the planner's call to make explicitly rather than leave implicit.

3. **Extended (hours-scale) crash-loop behavior**
   - What we know: 10 rapid crash cycles over 75 seconds produced no `launchd`-level disable.
   - What's unclear: Whether a much longer sustained crash-loop (hours/days, matching real-world "user leaves their Mac on with a broken engine for a week" scenarios) ever triggers any additional OS-level throttling or state beyond the steady 10-second `ThrottleInterval`.
   - Recommendation: This is exactly the "real verification step" D-05 calls for — schedule it as an explicit, longer-duration manual verification task in the plan, not something this research session can complete.

## Environment Availability

| Dependency | Required by | Available | Version | Fallback |
|------------|--------------|-----------|---------|----------|
| `launchctl` | BOOT-01, BOOT-02 | ✓ | Darwin 27.2 (arm64), `/bin/launchctl` | — (OS-native, always present on macOS 10.10+) |
| `plutil` | BOOT-01 (D-03) | ✓ | Darwin 27.2 (arm64), `/usr/bin/plutil` | — (OS-native, always present on macOS 10.2+) |
| `sfltool` | BOOT-02 diagnostic enrichment only | ✓ | Darwin 27.2 (arm64), `/usr/bin/sfltool` — undocumented/private | If absent on a given macOS version, detection falls back to `launchctl print`/`print-disabled` only (already the primary signal, per Pattern 3) |

**Missing dependencies with no fallback:** None.

## Validation Architecture

### Test Framework
| Property | Value |
|----------|-------|
| Framework | Node built-in test runner (`node:test`) — no third-party test framework `[VERIFIED: package.json "test": "node --test tests/*.test.mjs"]` |
| Config file | none — plain `node --test` invocation |
| Quick run command | `node --test tests/launchagent-detect.test.mjs` (new file — Wave 0 gap) |
| Full suite command | `npm test` (runs `node --test tests/*.test.mjs`) |

### Phase Requirements → Test Map

| Req ID | Behavior | Test type | Automated command | File exists? |
|--------|----------|-----------|--------------------|--------------|
| BOOT-01 | Plist generated passes `plutil -lint`; `bootstrap`/`bootout` round-trip cleanly | disruptive/integration (real OS side effects) | `node --test tests/disruptive/launchagent-install.test.mjs` | ❌ Wave 0 |
| BOOT-01 | Plist is installed to `~/Library/LaunchAgents/<Label>.plist` (not a transient path) | unit (path assertion) | `node --test tests/launchagent-detect.test.mjs` | ❌ Wave 0 |
| BOOT-02 | Detection logic correctly classifies not-installed / launchd-disabled / ok given mocked `launchctl` output | unit | `node --test tests/launchagent-detect.test.mjs` | ❌ Wave 0 |
| BOOT-02 | Real crash-loop does not falsely report "disabled" (matches this research's empirical finding) | manual/disruptive, longer-duration | manual verification per Open Question 3 | ❌ Wave 0 — manual-only, document as such |

### Sampling Rate
- **Per task commit:** `node --test tests/launchagent-detect.test.mjs`
- **Per wave merge:** `npm test` (full suite) — note `tests/disruptive/*.test.mjs` are excluded from the default `npm test` glob per the existing `test:disruptive` script split; run `node --test tests/disruptive/*.test.mjs` separately since it has real OS side effects (matches the project's existing convention for this directory).
- **Phase gate:** Full suite green, plus the manual reboot-survival check (Pitfall 2) and the manual extended crash-loop observation (Open Question 3), before `/gsd:verify-work`.

### Wave 0 Gaps
- [ ] `tests/disruptive/launchagent-install.test.mjs` — covers BOOT-01 (real `plutil`/`bootstrap`/`bootout` side effects, matches existing `tests/disruptive/` convention for tests with real system side effects)
- [ ] `tests/launchagent-detect.test.mjs` — covers BOOT-01 (install path assertion) and BOOT-02 (detection logic against mocked `launchctl` output)
- [ ] Framework install: none — `node:test` is already available via the Node runtime bundled with the project's existing test scripts.

## Security Domain

### Applicable ASVS Categories

| ASVS category | Applies | Standard control |
|----------------|---------|-------------------|
| V2 Authentication | No | No auth flow in this phase. |
| V3 Session Management | No | Not applicable. |
| V4 Access Control | No | Per-user LaunchAgent runs with exactly the logged-in user's own privileges — no elevation, no cross-user access, no `OperatorSid`-equivalent to misconfigure (D-03 explicitly notes this has no macOS equivalent because per-user LaunchAgents are inherently user-scoped). |
| V5 Input Validation | Yes | Every value interpolated into the plist template (install paths, log paths) originates from trusted, app-internal sources (Electron's own `app.getPath()`/packaged-resource paths), not external user input — but XML-escape any interpolated string defensively (a username or install path containing `&`/`<`/`>` would otherwise produce a plist that fails `plutil -lint`, which is itself the safety net D-03 already mandates). |
| V6 Cryptography | No | No secrets are stored or transmitted by this phase — a structural improvement over the Windows side, which handled `OperatorSid`/DPAPI-adjacent concerns that simply don't exist for a per-user LaunchAgent. |

### Known Threat Patterns for this stack

| Pattern | STRIDE | Standard mitigation |
|---------|--------|-----------------------|
| Plist XML injection via an unescaped interpolated path | Tampering | XML-escape all interpolated values when building the template string; `plutil -lint` catches malformed output as a secondary gate, but escaping is the primary control. |
| Privilege confusion (assuming this LaunchAgent runs with elevated rights like the old Windows SYSTEM-scoped task) | Elevation of Privilege | Per-user LaunchAgents in `~/Library/LaunchAgents` always run as the logged-in user, never as `root` or SYSTEM — do not port any of the Windows side's SID/ACL-protection logic (`Assert-BootProtectedAcl`, `Assert-BootRegistryKey` in `tools/boot-engine-task.ps1`), it has no macOS analog and is not needed. |

## Sources

### Primary (HIGH confidence)
- `man launchd.plist(5)` — read directly on this machine (Darwin, dated 30 July 2019 in the man page footer), all plist keys quoted verbatim above (`Label`, `Disabled`, `ProgramArguments`, `RunAtLoad`, `KeepAlive`/`SuccessfulExit`/`ThrottleInterval`, `StandardOutPath`/`StandardErrorPath`, `WorkingDirectory`, `EnvironmentVariables`, FILES section).
- `man launchctl(1)` — read directly on this machine, `bootstrap`/`bootout`/`enable`/`disable`/`print`/`print-disabled` subcommand definitions quoted verbatim above.
- `man plutil(1)` — read directly on this machine, `-lint` option definition.
- Live empirical test, this session (2026-09-17, Darwin 27.2 arm64): bootstrapped a real crash-looping (`exit 1`, `KeepAlive=true`) LaunchAgent, observed `launchctl print` output and `log show` entries over 75+ seconds (10 respawn cycles, all throttled at ~10s, never disabled), confirmed absence from `launchctl print-disabled`, then cleanly booted it out and confirmed removal via `launchctl print` returning "Could not find service."
- Live empirical test, this session: `plutil -lint` against both a valid and a deliberately truncated plist, confirming exit 0/"OK" vs exit 1/"Encountered unexpected EOF."
- `sfltool dumpbtm` — run live on this machine; observed real per-user `~/Library/LaunchAgents` entries with a `Disposition` field (`enabled/disabled`, `allowed/disallowed`, `notified/not-notified`). This tool is undocumented by Apple (no man page) — treated as MEDIUM-confidence enrichment, not a primary API, despite being empirically run.
- `docs/engine-contract.md:15-16` (this repo) — boot-host ownership model and the exit-code shutdown contract, read directly this session.
- `apps/desktop/main/index.ts:1275`, `tools/boot-engine-task.ps1:85-90` (this repo) — verified Windows-side path precedents, read directly this session.
- `package.json:34` (this repo) — `appId: "com.maurdekye.orgtree"`, read directly this session.

### Secondary (MEDIUM confidence)
- WebSearch results cross-checked against the primary man pages above (`launchctl bootstrap`/`bootout` domain-target syntax, `RunAtLoad`/`KeepAlive` semantics) — used only to triangulate before going to the primary source; the man page quotes are what's cited in-text.
- MacRumors/Apple Community reporting on Ventura-era "Background Items Added" notification bugs and user-toggleable "Allow in the Background" list — used to corroborate that BTM disable is user/OS-toggleable and not reversible via `launchctl`.

### Tertiary (LOW confidence)
- `python-build-standalone` `install_only` tarball directory layout (`python/bin/python3`) — sourced from a GitHub-adjacent/blog summary (gregoryszorc.com, deepwiki), not astral-sh's own documentation. Logged as Assumption A1.

## Metadata

**Confidence breakdown:**
- Standard stack (launchctl/plutil usage): HIGH — primary man pages read locally plus live empirical reproduction of the full install/observe/uninstall cycle.
- Crash-loop/disabled detection design: HIGH for what `launchd` itself does (empirically reproduced); MEDIUM for the BTM/`sfltool` layer (undocumented tool, short observation window) — logged as Assumptions A2/A3.
- Packaged Python interpreter path for `ProgramArguments`: LOW — Phase 1/2 have not executed in this worktree; flagged as Assumption A1 and Open Question 1.

**Research date:** 2026-09-17
**Valid until:** 90 days (OS-native CLI surface, `launchctl`/`plutil` semantics are stable across recent macOS releases; re-verify sooner if the target macOS minimum version changes)
