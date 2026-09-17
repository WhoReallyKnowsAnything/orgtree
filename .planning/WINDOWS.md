---
schema_version: 1
open_count: 1
waived_count: 0
fixed_count: 0
total_count: 1
last_updated: 2026-09-17T18:54:34.450Z
---

# Broken Windows Ledger

> Cross-phase defect register. With `workflow.windows_enforce` enabled, `/gsd-ship` blocks while `open_count > 0`.
> Waive with `gsd-tools windows waive <id> "<reason>"` (reason required).
> Mark fixed with `gsd-tools windows fixed <id>`.

| id | phase | kind | file | line | description | status | reason | recorded_at | resolved_at |
|----|-------|------|------|------|-------------|--------|--------|-------------|-------------|
| 1 | 02.1 | deviation | engine/process_lifetime.py |  | PosixTree.terminate() only provably sweeps a descendant in its own escaped process group (start_new_session=True, e.g. a provider CLI) when the guardian's sweep runs while the engine is still alive (parent-death or guardian-crash teardown). Once the engine itself exits or crashes first, the kernel reparents the escaped-group descendant away before the sweep enumerates, and it survives - a real gap against ROADMAP Phase 02.1 success criterion 2 for the common normal-exit path. | open |  | 2026-09-17T18:54:34.450Z |  |

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
  }
]
````
