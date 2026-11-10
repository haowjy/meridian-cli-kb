# Telemetry local persistence

Telemetry segments are per project so queries include events from both CLI and spawn processes. Compound owner and PID names allow spawn liveness to use spawn identity instead of trusting a recycled PID. See [telemetry decisions](../../decisions/telemetry.md).

A batch is the durability unit, with one tail repair, write and fsync; this avoids per-event I/O while leaving at most a torn final batch after a crash. See [batch durability](../../decisions/codebase-cleanup.md#fsync-once-per-batch-not-per-envelope).

The early CLI buffer upgrades in place to avoid leaving a background writer with a reference to the old sink.

