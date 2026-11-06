# Process-Scope Cleanup

See [process scope](../architecture/process-scope.md) and [future work](future-work.md#state--spawn).

## Open Items

### PROC-004: Dead Wrapper / Live Reparented Child Escapes Group Kill

A descendant that creates a new process group can escape the guardian's group signal after its parent dies. A process-table scan after reparenting cannot reconstruct the original tree. Stronger containment or a best-effort environment scan would have different platform and reliability costs.

### PROC-007: Audit Remaining Metadata-Only Lifecycle Edges

Audit whether background-worker or metadata-only paths can own long-lived processes without recording a scope; PID fallback provides weaker containment.

### PROC-009: `root_pid` Conflates Launcher, Health, and Containment Identity (issue #449)

A launcher shim can die while its serving child remains live in the recorded group. Group reachability improves diagnostics but is not verified process identity. Separate launcher, serving and containment identities remain open ([issue #449](https://github.com/haowjy/meridian-cli/issues/449)).

### PROC-008: Thread Windows Job Handles Through Scope Termination

The persisted snapshot cannot recreate a Job Object handle; explicit termination falls back to PID-tree cleanup after handle ownership is lost. Native Windows is not a supported design target.
