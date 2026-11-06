# Telemetry architecture

Telemetry supplies cross-subsystem diagnostics; lifecycle state and session records remain authoritative. Emission is best-effort, so consumers cannot rely on telemetry as a second state store.

Error reporting and feature tracking pipelines are not yet implemented. They are intended to read local segments rather than join the write path, keeping export and consent separate from the producer's failure budget. See [telemetry decisions](../../decisions/telemetry.md).

See [local persistence](local-persistence.md), [event catalog](event-catalog.md) and [reader and query](reader-and-query.md).

