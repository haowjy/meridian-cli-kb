# Codebase: Observability

Meridian's observability has two distinct layers:

1. **Structured logging and debug tracing** (`lib/observability/`) — transient,
   session-scoped, stderr-only.
2. **Telemetry spine** (`lib/telemetry/`) — persistent, queryable structured
   event log. Covers dead zones (chat transport, dev frontend, work transitions,
   etc.) and provides the foundation for error reporting and usage analytics.
   See [architecture/telemetry/overview.md](../architecture/telemetry/overview.md).

## Telemetry Spine

The telemetry spine fills coverage gaps that neither structlog nor the state layer reaches: chat HTTP/WS transport, dev frontend subprocess lifecycle, work item transitions, and runtime self-observability. It is the durable, queryable record that survives session boundaries.

See [../architecture/telemetry/overview.md](../architecture/telemetry/overview.md) for the three-layer model, sink protocol, per-project segment naming, and event taxonomy.

## Relationship to State Layer

Structured logs are **transient** — they help diagnose problems in the current session but are not persisted. The durable record is the state layer plus telemetry: for spawn state, read `spawns/<id>/state.json` and the spawn artifact directory (`history.jsonl`, `stderr.log`, `report.md`); for sessions, read `sessions.jsonl`; for observability dead zones, query telemetry segments.

## Related Pages

- [../architecture/telemetry/overview.md](../architecture/telemetry/overview.md) — telemetry spine: three-layer model, sink protocol, compound-named per-project segments, event taxonomy
- [../architecture/state-system.md](../architecture/state-system.md) — durable event record
- [../lessons/arch-refactor-pitfalls.md](../lessons/arch-refactor-pitfalls.md) — structlog `capture_logs()` cache pitfall encountered during the architecture refactor
