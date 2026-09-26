# Phase 3: Launchd Autostart - Context

**Gathered:** 2026-09-17 (from orchestrator dispatch + roadmap/requirements/research)
**Status:** Ready for planning

<domain>
## Phase Boundary

Replace the Windows Scheduled Task autostart mechanism (`tools/register-boot-engine.ps1`,
`tools/unregister-boot-engine.ps1`, `tools/boot-engine-task.ps1`) with a per-user macOS
`launchd` LaunchAgent. The engine must start automatically at login without user action, and
the app must detect and surface it when macOS has silently disabled the LaunchAgent after a
crash loop. Scope ends at autostart install/uninstall/detection — the `setLoginItemSettings`
GUI toggle (DIST-03) is a later, separate phase layered on top of this one.

</domain>

<decisions>
## Implementation Decisions

### Plist install/uninstall mechanism
- **D-01:** Use a per-user LaunchAgent plist written to `~/Library/LaunchAgents/`.
- **D-02:** Install/uninstall via the modern `launchctl bootstrap gui/<uid> <plist>` /
  `launchctl bootout gui/<uid> <plist>` verbs — not the deprecated `launchctl load`/`unload`.
- **D-03:** Validate the generated plist with `plutil -lint` before install, mirroring the
  `Prepare`/`Register`/`Remove`/`Stop` action shape of the existing Windows
  `Invoke-BootLifecycle` contract (`InstallDir`, `InstallMode` params carry over conceptually;
  `OperatorSid` has no macOS equivalent — per-user LaunchAgents are inherently user-scoped).

### Crash-loop detection
- **D-04:** macOS silently disables a LaunchAgent that crash-loops (repeated fast-exit) with
  no programmatic re-enable available. The app must detect this condition and prompt the user
  rather than failing to start with no explanation (BOOT-02). This is a first-class task, not
  an afterthought — it needs its own detection check (e.g. on next manual launch, check
  whether the LaunchAgent is loaded/running via `launchctl print gui/<uid>/<label>` and whether
  it was expected to have started this session) and a user-facing prompt/remediation path.

### Verification approach
- **D-05:** Research flags `launchctl bootstrap`/`bootout` usage as resting on secondary
  community sources, not primary Apple documentation. The plan must include a real verification
  step (actually install/uninstall a test LaunchAgent and observe `launchctl print` / process
  state) rather than trusting the research claims as-is.

### Claude's Discretion
- Exact plist key set (`Label`, `ProgramArguments`, `RunAtLoad`, `KeepAlive` shape, log paths)
  and exact crash-loop detection heuristic (which `launchctl print` fields / exit-code pattern
  to key off) are open for the planner/implementer to determine, as long as D-01 through D-05
  hold.

</decisions>

<specifics>
## Specific Ideas

No additional product-reference specifics beyond the roadmap success criteria and requirements
below — this phase is infrastructure/OS-integration, not user-facing UI.

</specifics>

<canonical_refs>
## Canonical References

**Downstream agents MUST read these before planning or implementing.**

### Requirements & roadmap
- `.planning/ROADMAP.md` § Phase 3: Launchd Autostart — phase goal, depends-on (Phase 1, 2),
  and the three success criteria this plan must satisfy.
- `.planning/REQUIREMENTS.md` (BOOT-01, BOOT-02) — the two requirements this phase closes.
  Note DIST-03 nearby references the LaunchAgent from BOOT-01 but is out of scope here.

### Research
- `.planning/research/SUMMARY.md` — confirms `launchd` (OS-native, no dependency) as the
  autostart mechanism, `launchctl bootstrap`/`bootout` (not legacy `load`/`unload`) as the
  install/uninstall verbs, and flags the bootstrap/bootout guidance as resting on secondary
  sources (see D-05).

### Existing Windows contract being replaced
- `tools/register-boot-engine.ps1`, `tools/unregister-boot-engine.ps1`,
  `tools/boot-engine-task.ps1` — current `Prepare`/`Register`/`Remove`/`Stop` lifecycle
  actions, `InstallDir`/`OperatorSid`/`InstallMode` parameters. The macOS plan replaces this
  entirely; it does not need to preserve the PowerShell interface, only the lifecycle
  guarantees (clean install, clean removal, correct operator/user scoping).

</canonical_refs>

<code_context>
## Existing Code Insights

### Reusable Assets
- None yet ported from Phase 1/2 review — planner should check Phase 1 (packaging) and
  Phase 2 (process lifecycle) plan/execution artifacts for the install-dir layout and Python
  engine entrypoint path this LaunchAgent's `ProgramArguments` must invoke.

### Established Patterns
- Windows side used a dedicated lifecycle function (`Invoke-BootLifecycle`) with explicit
  action verbs rather than inline scripting per call site — the macOS equivalent should keep
  install/uninstall/detect as discrete, independently testable operations.

### Integration Points
- Plist `ProgramArguments` must point at the packaged macOS engine binary/interpreter from
  Phase 1 (python-build-standalone bundle) and the process entrypoint from Phase 2.

</code_context>

<deferred>
## Deferred Ideas

- `setLoginItemSettings`-based GUI toggle for autostart (DIST-03) — explicitly deferred to a
  later phase per REQUIREMENTS.md, layered on top of this phase's LaunchAgent.

</deferred>
