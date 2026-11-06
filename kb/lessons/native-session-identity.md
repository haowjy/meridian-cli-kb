# Native Session Identity Lessons

## Green Suites Did Not Find the Seam Defects

Identity needs integrated verification as well as component tests: use decoy stores, identity-shaped prose, mismatched file contents, concurrent unrelated sessions, and real host lifecycle transitions. A fake can reproduce event order while missing the host's context invalidation. See [native session identity](../decisions/native-session-identity.md) and [review convergence](review-convergence-gate.md).

## Terminal Status Is Not Process Exit

A child may write exit evidence during teardown after the turn becomes terminal. Join the child before reading that evidence; polling for a file does not establish that teardown finished.

A completion marker for migration must distinguish a successful negative lookup from an unreadable source, or a temporary failure becomes permanent. SQLite read-only connections to WAL databases can still write shared-memory read marks; a snapshot is needed when zero writes are required. Repair must also clear any latched failure caused by the state it repairs. See [history storage](../decisions/history-storage.md).

A projection must share the authority's rule instead of reimplementing it. Real records agreeing do not prove equivalence: include deliberately divergent shapes. A diagnostic flag that changes an outcome needs every setter audited; a malformed event must not discard an independent authoritative total.

## A Test Helper That Reads Runner Bytes Trips the Blind Trap

A test helper is subject to the same read restrictions as the code it tests. A test mode also needs a positive assertion that it reached subprocesses: otherwise a silently disabled mode can produce deceptively better results.

## An In-Place Projection Upgrade Breaks the Build Still Running

A rebuildable index can still be live input to an older running build. Versioned index files let both builds coexist; see [history storage](../decisions/history-storage.md#d-history-index-schema-namespace) and [upgrade probing](dogfooding-pr-builds.md#probing-the-upgrade-path-needs-a-genuinely-old-build-and-old-state).

An integrity digest covers persisted bytes or stored JSON, not a reserialization through an evolving model. See [portable digest decision](../decisions/history-storage.md#d-history-portable-digest-stored-json).

Recovery inference must measure wrong bindings as well as yield: a wrong immutable binding is permanent. When a heuristic offers an action, constrain what it can emit under uncertainty before improving the guess; a placeholder is safer than a runnable command with an unknown endpoint.
