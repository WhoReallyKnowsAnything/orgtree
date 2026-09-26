# Requirements: Orgtree (macOS fork)

**Defined:** 2026-09-26
**Core Value:** The app runs and works correctly on macOS without breaking the Windows build that upstream ships.

## Active

Committed scope. Each maps to exactly one roadmap phase.

### Hardening

- [ ] **HARD-01**: Developer can run the full `npm test` suite on macOS with zero failures, including `tests/update-rehearsal.test.mjs`
- [ ] **HARD-02**: User who reboots their Mac finds the Orgtree engine running without opening the app
- [ ] **HARD-03**: User whose engine crash-loops sees the remediation dialog, not a false "disabled" state

### Upstream Sync

- [ ] **SYNC-01**: User can build and launch the fork on macOS from a tree that contains current `upstream/main`
- [ ] **SYNC-02**: User can complete a real agent turn on macOS after the upstream merge
- [ ] **SYNC-03**: Windows user can build and install the fork's NSIS installer after the upstream merge

### Distribution

- [ ] **DIST-01**: User can download a signed, notarized `.dmg` and open Orgtree without a Gatekeeper override
- [ ] **DIST-02**: User on an older macOS release is offered and can install a newer one through the in-app updater
- [ ] **DIST-03**: User can turn engine autostart on or off from the app's settings

## v2 Requirements

Deferred. Tracked, not in the current roadmap.

(None yet)

## Out of Scope

| Feature | Reason |
|---------|--------|
| Linux support | Not requested; no Linux-specific work in the codebase |
| `.pkg` installer | `.dmg` covers distribution |
| Framework rewrite | Would make the fork unmergeable with upstream |

## Traceability

| Requirement | Phase | Status |
|-------------|-------|--------|

---
*Last updated: 2026-09-26 after /cad-adopt*
