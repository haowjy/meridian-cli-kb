# Telemetry: Local Persistence

`LocalJSONLSink` is the always-on write path for v1 telemetry. Each process writes to its own flat JSONL segment file under the project's runtime root. The `BufferingSink` covers the window before the runtime root is known.

**Related:**
- [overview.md](overview.md) — three-layer model, write path diagram
- [reader-and-query.md](reader-and-query.md) — how segments are read back
- [decisions/telemetry.md](../../decisions/telemetry.md) — D62-tel (flat vs partitioned), D66 (compound naming), D67 (per-project storage), D68 (BufferingSink upgrade)

---

## Sink Protocol

```python
class TelemetrySink(Protocol):
    def write_batch(self, events: Sequence[TelemetryEnvelope]) -> None:
        """Write a batch of events. Must not raise."""
        ...

    def close(self) -> None:
        """Flush and release resources. Idempotent."""
        ...
```

`write_batch` operates on batches, not individual events. The background writer accumulates events and calls `write_batch` on the flush interval (every 1 second or 100 events, whichever comes first). This avoids per-event I/O overhead.

Defined in `src/meridian/lib/telemetry/sinks.py`.

---

## LocalJSONLSink

`LocalJSONLSink` in `src/meridian/lib/telemetry/local_jsonl.py` writes compact JSON lines to a per-process segment file.

### Initialization

```python
class LocalJSONLSink:
    def __init__(
        self,
        runtime_root: Path,
        *,
        max_segment_bytes: int = 10_000_000,
        logical_owner: str | None = None,
    ) -> None:
```

On construction it:
1. Creates `<runtime_root>/telemetry/` if it does not exist
2. Runs `run_retention_cleanup()` opportunistically
3. Scans existing segments to determine the next sequence number
4. Opens the initial segment file in append mode

### Segment Naming

Segment files use the compound format:

```
<logical_owner>.<pid>-<seq:04d>.jsonl
```

Examples:
```
<project_runtime_root>/telemetry/
  cli.12345-0001.jsonl          # CLI process, PID 12345, first segment
  cli.12345-0002.jsonl          # second segment after rotation
  chat.67890-0001.jsonl         # chat server segment
  p42.7812-0001.jsonl           # spawn p42, PID 7812
```

- `logical_owner`: `cli` (default), `chat`, or a spawn ID (e.g., `p42` from `MERIDIAN_SPAWN_ID`)
- `pid`: the OS process ID at time of construction
- `seq`: four-digit sequence number, increments on rotation

**Why compound names:** PID-only naming (`<pid>-<seq>.jsonl`) was insufficient because the OS recycles PIDs. `p42.7812-0001.jsonl` is live only if spawn `p42` is genuinely active — not merely if PID 7812 exists. See [decisions/telemetry.md#d66](../../decisions/telemetry.md#d66).

**Legacy format:** Files matching the old `<pid>-<seq>.jsonl` pattern (no dot separator) are parsed to `None` by `parse_segment_owner()` and treated as immediately orphaned by retention.

### Rotation

`write_batch()` checks segment size after each flush. When `active_path.stat().st_size > max_segment_bytes` (default 10 MB), it closes the current file and increments `_seq`, opening the next segment. The rotate is:

```python
def _rotate(self) -> None:
    if self._file is not None:
        self._file.flush()
        self._file.close()
        self._file = None
    self._seq += 1
    self._open_segment()
```

`_next_sequence()` scans existing segments matching `<logical_owner>.<pid>-*.jsonl` to find the highest used sequence number, so a restarted process that reuses the same PID doesn't overwrite a previous segment.

### Write Behavior

`write_batch()` writes each envelope as a compact JSON line (`separators=(",", ":")`). Lines that fail JSON serialization are silently skipped. `close()` flushes and closes the file handle; it is idempotent.

---

## Sink Selection by Process Type

`setup_telemetry()` in `src/meridian/lib/telemetry/init.py` picks the sink at process startup:

| Process type | Sink | Why |
|---|---|---|
| CLI (project root not yet resolved) | `BufferingSink` | Buffers early events; upgraded once root resolves |
| CLI or spawn with runtime root | `LocalJSONLSink` | Always-on; segments are queryable |
| MCP stdio server (rootless) | `StderrSink` | No runtime root; stdout is JSON protocol |
| Explicit opt-out | `NoopSink` | Discards all events |

`init_telemetry()` in `src/meridian/lib/telemetry/__init__.py` is the simpler variant used by non-CLI entry points: explicit sink wins; otherwise `LocalJSONLSink` if `runtime_root` is given; otherwise `NoopSink`.

---

## BufferingSink and the CLI Pre-Root Window

The CLI emits `usage.command.invoked` immediately at dispatch — before the project root is resolved. `BufferingSink` holds these early events and replays them once the root becomes known.

```mermaid
sequenceDiagram
    participant CLI
    participant Router as TelemetryRouter
    participant Buf as BufferingSink
    participant JSONL as LocalJSONLSink

    CLI->>Router: emit_telemetry("usage", "usage.command.invoked", ...)
    Router->>Buf: write_batch([envelope])
    Note over Buf: buffered in deque(maxlen=1000)

    CLI->>CLI: resolve project root
    CLI->>Buf: upgrade(LocalJSONLSink(runtime_root))
    Note over Buf: acquires lock, replays buffer to JSONL, switches sink in place

    CLI->>Router: emit_telemetry(...)
    Router->>Buf: write_batch([envelope])
    Note over Buf: real_sink is set; forwards directly to JSONL
```

`upgrade()` is an in-place operation on the `BufferingSink` object itself — it does not swap the router reference. This avoids a race where background threads hold a reference to the old router. See [decisions/telemetry.md#d68](../../decisions/telemetry.md#d68).

The upgrade is called from `upgrade_cli_telemetry_to_project()` in `src/meridian/cli/main.py`.

---

## Per-Project Storage

All telemetry segments — CLI and spawn — write to:

```
<project_runtime_root>/telemetry/
```

**Why per-project (not user-home):** Before this change, CLI wrote to `~/.meridian/telemetry/` while spawn processes already wrote per-project. The split made `meridian telemetry tail` miss events from one side or the other. Per-project colocation ensures `tail` and `query` see the full cross-process event stream without additional flags. See [decisions/telemetry.md#d67](../../decisions/telemetry.md#d67).

**Legacy segments:** `~/.meridian/telemetry/` (the old CLI write path) has no active writers. The `--global` flag includes this directory for reads so old segments remain visible until they age out naturally.

**Global reads:** `meridian telemetry tail/query/status --global` walks `~/.meridian/projects/*/telemetry/` at invocation time (point-in-time snapshot). See [reader-and-query.md](reader-and-query.md).

---

## Sink Implementations

| Class | Behavior | Location |
|---|---|---|
| `LocalJSONLSink` | Compound-named JSONL segments with rotation | `local_jsonl.py` |
| `BufferingSink` | Buffers events in `deque(maxlen=1000)`, upgrades in place | `sinks.py` |
| `NoopSink` | Discards all events | `sinks.py` |
| `StderrSink` | Compact JSON lines to stderr | `sinks.py` |
