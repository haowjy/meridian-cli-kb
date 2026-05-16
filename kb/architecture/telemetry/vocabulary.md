# Telemetry Vocabulary

Terms covering telemetry sinks, the event envelope, routing, and local persistence. For the full vocabulary index see [../../vocabulary.md](../../vocabulary.md).

| Term | Definition | See also |
|---|---|---|
| **Debug tracer** | A per-spawn structured JSONL logger (`DebugTracer`) that writes harness wire events to a per-spawn debug file. Best-effort; self-disables on first failure. Activated by `MERIDIAN_DEBUG=1` at the lifecycle observer level, not through this class. | [../../codebase/observability.md](../../codebase/observability.md) |
| **Local JSONL sink** | The primary telemetry sink (`LocalJSONLSink`). Writes JSONL segments into `<runtime_root>/telemetry/`. Segment naming: `<owner>.<pid>-<seq:04d>.jsonl`. Rotates at 10 MB. Runs retention cleanup on initialization. | [local-persistence.md](local-persistence.md) |
| **Telemetry envelope** | The v1 telemetry payload (`TelemetryEnvelope`). Fields: `v`, `ts`, `domain`, `event`, `scope`, optional `severity`, `ids`, `data`. `to_dict()` omits absent optional fields. | [event-catalog.md](event-catalog.md) |
| **Telemetry router** | The bounded-queue background writer (`TelemetryRouter`) distributing telemetry envelopes to sinks. Best-effort: never raises to callers; drops events on queue overflow and logs a synthetic `runtime.telemetry.dropped` envelope; disables failing sinks. | [overview.md](overview.md) |
| **Telemetry sink** | The protocol for telemetry output (`TelemetrySink`). Concrete implementations: `LocalJSONLSink` (file-backed), `BufferingSink` (buffers early events until upgraded), `NoopSink` (discard-all), `StderrSink` (compact JSON to stderr). | [overview.md](overview.md) |
