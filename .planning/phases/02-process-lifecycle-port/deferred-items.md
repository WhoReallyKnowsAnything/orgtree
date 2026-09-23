# Deferred Items — Phase 02 (process-lifecycle-port)

## 02-02: tests/test_mailhub_runtime.py — 6 pre-existing failures, out of scope

**Found during:** Task 2 regression check (`python -m unittest tests.test_mailhub_runtime -v`)

**Root cause:** `engine/mailhub` is a git submodule that is not checked out in
this worktree (`git submodule status` reports `-6e856eec8ccfbcf8d16451123903b9d2ce16450b
engine/mailhub`, and `ls engine/mailhub` is empty). Every test that spawns the
real `python -m mailhub.serve` child process fails with "the hub process
exited at startup (code 1)" because `mailhub.serve` cannot be imported.

**Confirmed unrelated to this plan's changes:** re-ran the same suite against
the unmodified (pre-fix) `engine/mailhub_runtime.py` via `git checkout HEAD --
engine/mailhub_runtime.py` — identical 8 passed / 6 FAILED result. The failure
predates and is independent of the `start_new_session`/`os.killpg` fix.

**Not fixed here:** initializing a git submodule is an environment/worktree
setup action outside this plan's scope (PROC-03 process-tree containment),
not a code defect in the files this plan touches.

**Failing tests (all pre-existing, all require the real submodule child):**
- a stale orphan pid file does not block a start
- a config change restarts the hub and persists
- an invalid patch is refused and the running hub is untouched
- (3 more with the same root cause — see full log)
