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

### PROC-007: Audit Remaining Metadata-Only Lifecycle Edges

**Where:** `src/meridian/lib/state/process_scope_projection.py`, background worker launch paths

**The issue:** Current managed backend and primary-attach paths record
`ProcessScopeSnapshot` entries in `process_scopes.json`, and `backend_lifecycle.json`
was removed. The remaining uncertainty is narrower: whether any background-worker or
metadata-only lifecycle path can still launch a long-lived process without recording a
scope sidecar. If such a path exists, cleanup falls back to recorded PIDs rather than
containment.

**What's needed:** Audit launch paths that do not go through `launch_managed_backend()`
or `PrimaryAttachLauncher` scope recording. Any path that can own a process tree should
record a `ProcessScopeSnapshot` before exposing the process.

**Why deferred:** The known managed-backend leak was fixed by centralizing launch and
recording scope snapshots. This item is now a coverage audit, not the primary cleanup
mechanism gap.

**Decision context:** [architecture/process-scope.md](../architecture/process-scope.md)

---
