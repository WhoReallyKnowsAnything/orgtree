---
phase: "3"
slug: "launchd-autostart"
status: draft
nyquist_compliant: false
wave_0_complete: false
created: "2026-09-17"
---

# Phase 3 — Validation Strategy

> Per-phase validation contract for feedback sampling during execution.

---

## Test Infrastructure

| Property | Value |
|----------|-------|
| **Framework** | Node built-in test runner (`node:test`) — no third-party test framework |
| **Config file** | none — plain `node --test` invocation |
| **Quick run command** | `node --test tests/launchagent-detect.test.mjs` |
| **Full suite command** | `npm test` (runs `node --test tests/*.test.mjs`; `tests/disruptive/*.test.mjs` excluded from this glob per existing convention) |
| **Estimated runtime** | ~5 seconds (quick), disruptive suite run separately |

---

## Sampling Rate

- **After every task commit:** Run `node --test tests/launchagent-detect.test.mjs`
- **After every plan wave:** Run `npm test` (full suite), plus `node --test tests/disruptive/*.test.mjs` separately since those have real OS side effects
- **Before `/gsd-verify-work`:** Full suite must be green, plus the manual reboot-survival check and the manual extended crash-loop observation (Open Question 3 in 03-RESEARCH.md)
- **Max feedback latency:** 5 seconds

---

## Per-Task Verification Map

| Task ID | Plan | Wave | Requirement | Threat Ref | Secure Behavior | Test Type | Automated Command | File Exists | Status |
|---------|------|------|-------------|------------|-----------------|-----------|-------------------|-------------|--------|
| 03-01-01 | 01 | 1 | BOOT-01 | — | Plist generated passes `plutil -lint`; `bootstrap`/`bootout` round-trip cleanly | disruptive/integration | `node --test tests/disruptive/launchagent-install.test.mjs` | ❌ W0 | ⬜ pending |
| 03-01-02 | 01 | 1 | BOOT-01 | — | Plist installed to `~/Library/LaunchAgents/<Label>.plist` (not a transient path) | unit | `node --test tests/launchagent-detect.test.mjs` | ❌ W0 | ⬜ pending |
| 03-02-01 | 02 | 1 | BOOT-02 | — | Detection logic classifies not-installed / launchd-disabled / ok given mocked `launchctl` output | unit | `node --test tests/launchagent-detect.test.mjs` | ❌ W0 | ⬜ pending |
| 03-02-02 | 02 | 1 | BOOT-02 | — | Real crash-loop does not falsely report "disabled" (matches empirical research finding) | manual/disruptive, longer-duration | manual verification per Open Question 3 | ❌ W0 — manual-only | ⬜ pending |

*Status: ⬜ pending · ✅ green · ❌ red · ⚠️ flaky*

---

## Wave 0 Requirements

- [ ] `tests/disruptive/launchagent-install.test.mjs` — covers BOOT-01 (real `plutil`/`bootstrap`/`bootout` side effects, matches existing `tests/disruptive/` convention)
- [ ] `tests/launchagent-detect.test.mjs` — covers BOOT-01 (install path assertion) and BOOT-02 (detection logic against mocked `launchctl` output)
- [ ] Framework install: none needed — `node:test` already available via the project's existing test scripts

---

## Manual-Only Verifications

| Behavior | Requirement | Why Manual | Test Instructions |
|----------|-------------|------------|-------------------|
| Real crash-loop does not trigger a false "disabled" report over hours-scale runtime | BOOT-02 | Research only observed ~75s of throttled crash-looping; BTM/`sfltool` disable behavior is undocumented and needs longer real-world observation | Install a deliberately crash-looping build, leave it running for several hours, confirm the app's detection logic doesn't misfire while `launchd` is still throttle-retrying, and separately confirm behavior once BTM actually disables the item (via `sfltool dumpbtm` / System Settings → Login Items) |
| Engine survives a real logout/login (reboot) cycle | BOOT-01 | Requires an actual macOS login session cycle, not mockable in CI | Log out and back in (or reboot), confirm the engine process is running without manual action |

---

## Validation Sign-Off

- [ ] All tasks have `<automated>` verify or Wave 0 dependencies
- [ ] Sampling continuity: no 3 consecutive tasks without automated verify
- [ ] Wave 0 covers all MISSING references
- [ ] No watch-mode flags
- [ ] Feedback latency < 5s
- [ ] `nyquist_compliant: true` set in frontmatter

**Approval:** pending
