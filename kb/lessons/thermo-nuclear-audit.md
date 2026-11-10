# Audit: Rejected Alternatives

A process-local ownership capability cannot secure mutations across a process boundary. Exactly-once cleanup gating also breaks crash recovery if a process dies after recording the gate but before terminating the target. A durable claim and retryable, identity-validated cleanup preserve recovery; see [state decisions](../architecture/state-system/overview.md).

A generic JSON helper can erase meaningful distinctions: an in-memory SQLite value and a file read that distinguishes missing from unreadable do not have the same contract. Likewise, a configuration parser replacement must preserve merging, diagnostics, and coercion behavior, not just accept the same syntax.

Raw harness events remain open because unpinned harnesses can add event types; a closed normalized semantic layer still gives shared code a stable vocabulary. See [state decisions](../architecture/state-system/overview.md).
