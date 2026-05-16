# Telemetry: Event Catalog

Every telemetry event conforms to the `TelemetryEnvelope` schema and must be registered in `EVENT_REGISTRY`. This page covers the envelope shape, the six event domains, concern classification, and validation rules.

**Related:**
- [overview.md](overview.md) — write path, three-layer model
- [reader-and-query.md](reader-and-query.md) — how concern and domain filter reads
- [decisions/telemetry.md](../../decisions/telemetry.md) — D63-tel (concern in registry), D64 (8-field envelope)

Source: `src/meridian/lib/telemetry/events.py`

---

## TelemetryEnvelope

```python
@dataclass(frozen=True)
class TelemetryEnvelope:
    v: int                          # envelope version (currently 1)
    ts: str                         # UTC ISO-8601 with Z suffix
    domain: str                     # one of six valid domains
    event: str                      # stable event name from EVENT_REGISTRY
    scope: str                      # emitter identity as dotted path
    severity: str | None = None     # debug | info | warning | error (default info)
    ids: dict[str, str] | None = None   # correlation keys
    data: dict[str, Any] | None = None  # event-specific payload
```

`to_dict()` omits absent optional fields (`severity`, `ids`, `data`) to minimize per-record overhead. A minimal envelope serializes to 5 fields.

### Field reference

| Field | Required | Values / format | Purpose |
|---|---|---|---|
| `v` | yes | `1` | Schema version guard |
| `ts` | yes | `2026-05-02T12:00:00Z` | Chronological ordering, retention checks |
| `domain` | yes | `spawn`, `chat`, `server`, `work`, `runtime`, `usage` | Coarse filter, maps to domain pages |
| `event` | yes | Dotted name, e.g. `chat.ws.connected` | Stable event identity — must exist in `EVENT_REGISTRY` |
| `scope` | yes | Dotted path, e.g. `chat.server.ws` | Emitter location for debugging |
| `severity` | no | `debug`, `info`, `warning`, `error` | Triage signal |
| `ids` | no | `{"spawn_id": "p42", "chat_id": "c-abc"}` | Foreign keys for joining to other stores |
| `data` | no | Event-specific dict | Payload — schema is per-event, not validated by the router |

**An example envelope:**

```json
{
  "v": 1,
  "ts": "2026-05-02T12:00:00Z",
  "domain": "chat",
  "event": "chat.ws.connected",
  "scope": "chat.server.ws",
  "ids": {"chat_id": "c-abc123"}
}
```

The 14-field earlier proposal was simplified to 8 fields. Six fields were removed or deferred (including `observed_ts`, `resource`, `consent`, `schema`, `body`, `seq`). See [decisions/telemetry.md#d64](../../decisions/telemetry.md#d64).

---

## Event Domains

Six domains partition the event namespace:

### `spawn`

Sparse lifecycle correlation markers — only terminal events. Full spawn lifecycle lives in per-spawn `state.json` plus spawn artifacts.

| Event | Concerns | `ids` keys |
|---|---|---|
| `spawn.process_exited` | `operational` | `spawn_id` |
| `spawn.succeeded` | `operational` | `spawn_id` |
| `spawn.failed` | `operational`, `error` | `spawn_id` |
| `spawn.cancelled` | `operational` | `spawn_id` |

`spawn.failed` carries structured error data via `make_error_data()`: exception type, message, and stack trace.

### `chat`

Dead-zone events for the chat transport layer. Previously silent.

| Event | Concerns | `ids` keys |
|---|---|---|
| `chat.http.request_completed` | `operational` | `chat_id`, `command_id` |
| `chat.ws.connected` | `operational`, `usage` | `chat_id` |
| `chat.ws.disconnected` | `operational` | `chat_id` |
| `chat.ws.rejected` | `operational`, `error` | `chat_id` |
| `chat.command.dispatched` | `operational`, `usage` | `chat_id`, `command_id` |
| `chat.runtime.stopping` | `operational` | `chat_id` |
| `chat.runtime.stopped` | `operational` | `chat_id` |

### `server`

Dev frontend subprocess and MCP command lifecycle.

| Event | Concerns | `ids` keys |
|---|---|---|
| `dev_frontend.launched` | `operational`, `usage` | — |
| `dev_frontend.ready` | `operational`, `usage` | — |
| `dev_frontend.readiness_timeout` | `operational`, `error` | — |
| `dev_frontend.exited` | `operational` | — |
| `mcp.command.invoked` | `operational`, `usage` | `request_id`, `work_id`, `spawn_id` |

Note: `dev_frontend.*` events use domain `server` despite the event names using the `dev_frontend.` prefix.

### `work`

Work item lifecycle transitions.

| Event | Concerns | `ids` keys |
|---|---|---|
| `work.started` | `operational`, `usage` | `work_id` |
| `work.updated` | `operational` | `work_id` |
| `work.done` | `operational`, `usage` | `work_id` |
| `work.deleted` | `operational` | `work_id` |
| `work.reopened` | `operational`, `usage` | `work_id` |
| `work.renamed` | `operational` | `work_id` |

### `runtime`

Telemetry self-observation and internal failure modes.

| Event | Concerns | `ids` keys |
|---|---|---|
| `runtime.telemetry.dropped` | `operational`, `error` | — |
| `runtime.telemetry.sink_failed` | `operational`, `error` | — |
| `runtime.telemetry.consumer_data_lost` | `operational`, `error` | `consumer_id` |
| `runtime.debug_tracer_disabled` | `operational`, `error` | — |
| `runtime.stream_event_dropped` | `operational`, `error` | `spawn_id` |

`runtime.telemetry.dropped` is a synthetic event emitted when the bounded queue overflows — the drop counter is embedded in `data`. `runtime.telemetry.sink_failed` is emitted once when a sink fails; the sink is then disabled.

### `usage`

Feature adoption signals as feedstock for the v3 analytics pipeline.

| Event | Concerns | `ids` keys | Call site |
|---|---|---|---|
| `usage.command.invoked` | `usage` | — | `cli/main.py` |
| `usage.model.selected` | `usage` | `spawn_id` | `launch/context.py` |
| `usage.spawn.launched` | `usage` | `spawn_id` | `ops/spawn/api.py` |

---

## Concern Classification

Every event has one or more concern tags: `operational`, `error`, or `usage`. Concerns are **not serialized** in each event record — they are a static mapping in `EVENT_REGISTRY`.

```python
VALID_CONCERNS = frozenset({"operational", "error", "usage"})

def concerns_for_event(event: str) -> tuple[Concern, ...]:
    return EVENT_REGISTRY[event]["concerns"]
```

Readers resolve concern from the event name at O(1) cost. This keeps records small and prevents per-emitter divergence. Multi-concern events are expressed naturally:

```python
"spawn.failed": {"domain": "spawn", "concerns": ("operational", "error"), "ids": ("spawn_id",)},
"chat.ws.connected": {"domain": "chat", "concerns": ("operational", "usage"), "ids": ("chat_id",)},
```

See [decisions/telemetry.md#d63-tel](../../decisions/telemetry.md#d63-tel) for why a per-record `concern` field was rejected.

---

## Validation

`validate_event(domain, event, severity)` in `events.py` is the router's intake gate:

```python
def validate_event(domain: str, event: str, severity: str | None) -> None:
    if domain not in VALID_DOMAINS:
        raise ValueError(f"invalid telemetry domain: {domain}")
    if severity is not None and severity not in VALID_SEVERITIES:
        raise ValueError(f"invalid telemetry severity: {severity}")
    definition = EVENT_REGISTRY.get(event)
    if definition is None:
        raise ValueError(f"unknown telemetry event: {event}")
    if definition["domain"] != domain:
        raise ValueError(...)
```

Validation is intentionally narrow: it checks domain membership, severity membership, and event-domain consistency. It does **not** validate `ids` or `data` shape — those are event-specific and not enforced at the router level.

---

## Error Data Helper

`make_error_data()` builds the structured error payload for `error`-concern events:

```python
def make_error_data(
    exc: BaseException | None = None, *, message: str | None = None
) -> dict[str, Any]:
```

Returns a dict with an `"error"` key containing:
- `type` — exception class name (or `"UnknownError"`)
- `message` — string message
- `stack` — full traceback when an exception object is provided

Callers pass this dict as `data=` to `emit_telemetry()`.

---

## EventDefinition Schema

Each entry in `EVENT_REGISTRY` is an `EventDefinition` TypedDict:

```python
class EventDefinition(TypedDict):
    domain: Domain          # which domain the event belongs to
    concerns: tuple[Concern, ...]   # one or more concern tags
    ids: tuple[str, ...]    # expected correlation key names (informational)
```

The `ids` field lists the expected keys as a documentation contract — the router does not enforce `ids` contents at runtime. New events must be added to `EVENT_REGISTRY` before they can be emitted; `validate_event()` rejects unknown events.
