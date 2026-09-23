---
schema_version: 1
open_count: 2
waived_count: 0
fixed_count: 0
total_count: 2
last_updated: 2026-09-23T10:17:01.689Z
---

# Broken Windows Ledger

> Cross-phase defect register. With `workflow.windows_enforce` enabled, `/gsd-ship` blocks while `open_count > 0`.
> Waive with `gsd-tools windows waive <id> "<reason>"` (reason required).
> Mark fixed with `gsd-tools windows fixed <id>`.

| id | phase | kind | file | line | description | status | reason | recorded_at | resolved_at |
|----|-------|------|------|------|-------------|--------|--------|-------------|-------------|
| 1 | 02.1 | deviation | engine/process_lifetime.py |  | PosixTree.terminate() only provably sweeps a descendant in its own escaped process group (start_new_session=True, e.g. a provider CLI) when the guardian's sweep runs while the engine is still alive (parent-death or guardian-crash teardown). Once the engine itself exits or crashes first, the kernel reparents the escaped-group descendant away before the sweep enumerates, and it survives - a real gap against ROADMAP Phase 02.1 success criterion 2 for the common normal-exit path. | open |  | 2026-09-17T18:54:34.450Z |  |
| 2 | 02 | deviation | tests/ |  | unittest discover -s tests -p 'test_*.py' shows pre-existing cross-test DATA_ROOT-binds-at-import-time contamination (91 fail/478 err) unrelated to 02-01's codexrun.py/antigravityrun.py changes; both named regression-guard files pass in isolation. See phases/02-process-lifecycle-port/deferred-items.md | open |  | 2026-09-23T10:17:01.689Z |  |

````json
[
  {
    "id": 1,
    "kind": "deviation",
    "phase": "02.1",
    "file": "engine/process_lifetime.py",
    "line": null,
    "description": "PosixTree.terminate() only provably sweeps a descendant in its own escaped process group (start_new_session=True, e.g. a provider CLI) when the guardian's sweep runs while the engine is still alive (parent-death or guardian-crash teardown). Once the engine itself exits or crashes first, the kernel reparents the escaped-group descendant away before the sweep enumerates, and it survives - a real gap against ROADMAP Phase 02.1 success criterion 2 for the common normal-exit path.",
    "status": "open",
    "reason": "",
    "recorded_at": "2026-09-17T18:54:34.450Z",
    "resolved_at": null
  },
  {
    "id": 2,
    "kind": "deviation",
    "phase": "02",
    "file": "tests/",
    "line": null,
    "description": "unittest discover -s tests -p 'test_*.py' shows pre-existing cross-test DATA_ROOT-binds-at-import-time contamination (91 fail/478 err) unrelated to 02-01's codexrun.py/antigravityrun.py changes; both named regression-guard files pass in isolation. See phases/02-process-lifecycle-port/deferred-items.md",
    "status": "open",
    "reason": "",
    "recorded_at": "2026-09-23T10:17:01.689Z",
    "resolved_at": null
  }
]
````
