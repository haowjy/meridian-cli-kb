# Open Questions: Process-Scope Cleanup

Process-scope cleanup gaps that remain after the durable process-scope ownership design. These items share one mechanism boundary: containment and recovery for subprocess trees that may outlive their owning spawn.

Related pages:
- [architecture/process-scope.md](../architecture/process-scope.md) — ownership model, scope projection, platform adapters, and cleanup policy
- [decisions/launch.md](../decisions/launch.md#d-process-scope-ownership) — decision rationale for durable process-scope ownership
- [future-work.md](future-work.md#state--spawn) — broader state/spawn deferred-work index

---

## Open Items

### PROC-004: Dead Wrapper / Live Reparented Child Escapes Group Kill

**Where:** `src/meridian/lib/platform/process_scope/posix.py`, reaper path

**The issue:** When an intermediate process exits before the scope snapshot is taken,
its children reparent to PID 1 and start a new process group. A subsequent `os.killpg`
against the original group PGID will not reach them. The psutil snapshot taken during
scope setup also misses them for the same reason.

**Observed scenario:** A harness wrapper forks a helper that itself forks a long-lived
tool worker. If the helper exits promptly, the tool worker is in the init group and
invisible to both the PGID kill and the psutil tree walk rooted at the helper.

**What's needed:** Either (a) a deeper containment mechanism (cgroup on Linux, fully
assigned Job Object on Windows covering all descendants at creation time), or (b) a
post-kill audit pass that checks for living processes with `MERIDIAN_SPAWN_ID` in their
env or cmdline. Option (a) requires OS-specific capabilities not yet scaffolded;
option (b) is heuristic.

**Why deferred:** Group kill handles the common case. The double-reparent scenario
requires harnesses that fork through disposable intermediaries, which is uncommon in
practice. Defer until there is a concrete observed leak of this type.

**Decision context:** [decisions/launch.md](../decisions/launch.md#d-process-scope-ownership)
and [architecture/process-scope.md](../architecture/process-scope.md)

---

### PROC-007: Metadata-Only Lifecycle Spawns Not Covered by Scope Projection

**Where:** `src/meridian/lib/state/process_scope_projection.py`, background worker launch paths

**The issue:** Background-worker spawns that use only lifecycle events (no spawn-state
`process_scopes` field) are not covered by the scope projection. The reaper still uses
`worker_pid` single-PID termination for these spawns. They do not benefit from group
kill or Job Object cleanup.

**What's needed:** Teach the worker launch path (the path that emits only lifecycle
events, not a full `state.json`) to emit scope events before starting the subprocess.
This requires threading a scope snapshot into the lifecycle event emitter and persisting
it before the subprocess starts.

**Why deferred:** These spawns are typically short-lived background workers whose cleanup
is less critical. The risk is lower than for harness-launched subtrees with tool children.
Deferred until the worker launch path is refactored for other reasons.

**Decision context:** [architecture/process-scope.md](../architecture/process-scope.md)

---
