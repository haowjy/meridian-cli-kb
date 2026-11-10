# Cost Cuts That Weakened Safety Checks

Before narrowing a scan, wait, or poll, identify the guarantees it provided and check that the faster path preserves them. A test proving work disappeared cannot prove that a safety refusal still works.

## A faster wake can change arbitration order

Reducing latency can change a priority-ordered race: a task that merely woke must not win over the task that caused the outcome. Check each `FIRST_COMPLETED` race when changing wake timing; see [terminal arbitration](../architecture/spawn-finalization.md#terminal-arbitration).

A keyed lookup replaces a scan only if every writer of the fact updates that key. Graph checks must follow every relevant edge, not a convenient representative. A disposable projection needs an actual rebuild path; exercise deletion with gaps in its source state. For ordering equivalence, include ties, backward clocks, interleaved writers, and reused keys. See [spawn identity](../architecture/state-system/spawn-launches.md#one-public-identity-launches-stay-internal) and [state system](../architecture/state-system/overview.md).
