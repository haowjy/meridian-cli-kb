# State roots and work items

A work directory's location determines whether it is active or archived, so interrupted moves can be retried without treating stale metadata as authority. Per-item links let running executions follow the directory as it moves. See [work and checkout links](../../decisions/session-work-links.md) and the source checkout's `src/meridian/lib/state/.context/CONTEXT.md`.

Work metadata travels between builds. Readers preserve unknown fields and refuse a newer format on mutation, protecting information written by newer builds. See [compatibility versioning](../../decisions/compatibility-versioning.md#d-staged-version-add--readers-learn-a-version-one-release-before-writers-write-it).

