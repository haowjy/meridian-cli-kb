# Decisions: Telemetry

Decision records for the telemetry system — why this architecture, what alternatives were rejected, what constraints drove the choices.

See [../decisions.md](../decisions.md) for the chronological index. Architecture mechanism is documented in [../architecture/telemetry/](../architecture/telemetry/).

---

## Background: Why Telemetry Was Added

The system was built after a chat WebSocket disconnect investigation (session c2926) revealed that the entire chat backend was silent — no request logs, no connect/disconnect traces, no frontend subprocess lifecycle events. The "dead zones" (areas with zero observability) were extensive enough that diagnosing even basic failures required adding instrumentation just to understand what was failing.

The design constraint: fill these dead zones without duplicating data that already lives in existing durable stores (`spawns/<id>/state.json`, `sessions.jsonl`, `chat/<id>/history.jsonl`).

---

## Architecture: Layers and Pipelines

### D61-tel: v2/v3 layers are readers, not sinks {#d61-tel}

*2026-05-02*

**Decided:** Error reporting (v2) and feature tracking (v3) consume telemetry by **reading** local JSONL segments with a persisted cursor. They are not `TelemetrySink` implementations sitting on the hot write path.

**Alternatives compared:**

| Option | Description | Rejected reason |
|---|---|---|
| A: v2+v3 as sinks | Register alongside `LocalJSONLSink` on write path | Couples later concerns to the write path; reintroduces backpressure questions into v1; contradicts requirements ("pipelines read from the local log") |
| B: All as readers | Chosen | Preserves v1's failure budget; consent boundaries stay in reader components; fits analytics (periodic aggregation, not per-event fan-out); near-real-time error reporting still achievable via segment follower with durable cursor |
| C: Error as sink, analytics as reader | Split architecture | Error latency marginally better, but inconsistency across v2/v3 adds surface area without commensurate benefit; v1 handles latency concern by making error events self-describing at emit time |

**Why this matters:** v1's failure budget is non-blocking, fire-and-forget: `emit_telemetry()` never blocks. Adding sinks on the write path would reintroduce backpressure and failure propagation for v2/v3 requirements that don't need it.

---

### D65: Telemetry complements existing stores, does not duplicate them {#d65}

*2026-05-02*

**Decided:** The telemetry stream carries only new dead-zone events and sparse spawn lifecycle correlation markers. Full spawn lifecycle, session events, and chat turn data stay in their existing durable stores. Cross-domain queries JOIN at read time via `ids`.

**What flows through telemetry:**
- New dead-zone events (chat transport, command dispatch, runtime stop, dev frontend, MCP commands, work transitions)
- Sparse lifecycle correlation markers — only terminal spawn events (`spawn.succeeded`, `spawn.failed`, `spawn.cancelled`) and `spawn.process_exited`
- Runtime self-observability (drop counters, sink failures, stream backpressure)

**What stays in existing stores (not duplicated):**
- Full spawn lifecycle → `spawns/<id>/state.json` plus spawn artifacts
- Session start/stop/update → `sessions.jsonl`
- Chat turns and content → `chat/<id>/history.jsonl`

**Why:** Projecting every `ChatEvent` and `LifecycleEvent` into the telemetry stream would duplicate already-durable data, violate the proportionality constraint (telemetry must be smaller than what it observes), and create synchronization problems between two stores of the same data.

---

## Storage Layout

### D62-tel: Flat per-process segment files over domain-partitioned directories {#d62-tel}

*2026-05-02 — naming superseded by D66, storage location superseded by D67*

**Decided:** One file per process, flat under the telemetry directory. Domain-partitioned directories were rejected.

**Why flat:**
- Orphan detection is trivial — check if the owning process is live.
- No date-directory cleanup races (crash-only retention runs at startup, not continuously).
- Read path is simple — glob `*.jsonl`, filter in-memory by `domain` field.
- Events interleave chronologically, which is what `tail` naturally wants.

**Why not domain-partitioned:** Domain filtering happens at read time via the `domain` field in each event. Pre-partitioned directories benefit only the write path, which writes one event at a time anyway. The read path benefits more from chronological interleaving than from pre-sorted buckets.

---

### D66: Compound segment naming `<logical_owner>.<pid>-<seq>.jsonl` {#d66}

*2026-05-02*

**Decided:** Segment filenames use the compound format `<logical_owner>.<pid>-<seq>.jsonl` where `logical_owner` is `cli`, `chat`, or a spawn ID (e.g. `p42`). Pure `<pid>-<seq>.jsonl` naming is treated as legacy (parsed to `None` by `parse_segment_owner()`, immediately orphaned).

**Why PID-only was insufficient:**

1. **PID reuse:** The OS recycles PIDs. `12345-0001.jsonl` from a dead spawn could be confused with a new unrelated process if that PID was recycled. The logical owner disambiguates: `p42.12345-0001.jsonl` is only live if spawn `p42` is genuinely active, not merely if PID 12345 exists.
2. **Same-PID ambiguity:** In theory a CLI process and a spawn process could share the same PID across machine restarts. Owner prefix makes each segment's identity unambiguous.

**Deferred-rename approach rejected:** Write to a temp file, rename to compound name on clean close. Rejected because: (1) contradicts crash-only model — events written before a crash would have no stable filename and be unrecoverable; (2) adds complexity to reader discovery. Compound-name-at-open is simpler and append-safe (each flushed line is durable regardless of clean close).

---

### D67: Per-project storage under `<project_runtime_root>/telemetry/` {#d67}

*2026-05-02*

**Decided:** All telemetry segments (CLI and spawn processes) write to `<project_runtime_root>/telemetry/`.

**The problem this solves:** Before this change, the CLI wrote to user-home (`~/.meridian/telemetry/`) while spawn processes already wrote per-project. The split made `meridian telemetry tail` miss CLI events when pointed at the project root, and miss spawn events when pointed at user-home. Cross-process queries were silently incomplete.

**Fix mechanism:** CLI uses `BufferingSink` → upgraded to per-project `LocalJSONLSink` once the root resolves (see [D68](#d68)). Spawn processes already wrote per-project; no change needed.

**Cross-project reads:** `--global` walks all project runtime roots (see [D70](#d70)). Global CLI operations that run outside a project fall back to `--global` with a message directing the user.

---

### D68: BufferingSink for pre-root CLI events; in-place upgrade avoids router-swap race {#d68}

*2026-05-02*

**Decided:** The CLI installs a `BufferingSink` at process start before the project root is resolved. Once the root is known, `upgrade_cli_telemetry_to_project()` atomically replays the buffer into a `LocalJSONLSink`, switching the sink in place on the same object.

**Why not deferred-create:** Deferring the router's existence until the root is known would lose events emitted before root resolution — including `usage.command.invoked`, which fires immediately at dispatch for every CLI invocation. That event is one of the most important usage signals for v3 feature tracking.

**Why in-place upgrade over router swap:** Swapping the global router reference (replacing `TelemetryRouter(BufferingSink)` with `TelemetryRouter(LocalJSONLSink)`) has a race: background threads may hold a reference to the old router and continue to write there after the swap. In-place upgrade keeps the same router and the same `BufferingSink` object; the upgrade method acquires the sink's lock, replays buffered events, and switches atomically.

---

## Event Schema

### D63-tel: Concern classification via event registry, not per-record field {#d63-tel}

*2026-05-02*

**Decided:** Concern tags (`operational`, `error`, `usage`) are a static mapping from event name to tags in the normative `EVENT_REGISTRY` in `lib/telemetry/events.py`. They are not serialized in every event record.

**Why a per-record `concern` field was rejected:**
- Adds ~40 bytes overhead per event.
- Concern is fixed by event definition, not by the emitter at runtime.
- The registry handles multi-concern events (e.g., `spawn.failed` is both `operational` and `error`) without per-record redundancy.
- Readers already know the event name — resolving concern from a static map is O(1) with no additional parse cost.

---

### D64: Simplified 8-field telemetry envelope {#d64}

*2026-05-02*

**Decided:** The canonical `TelemetryEnvelope` has 8 fields: `v`, `ts`, `domain`, `event`, `scope`, `severity`, `ids`, `data`. A 14-field earlier proposal was rejected.

**What was removed and why:**

| Field | Removed reason |
|---|---|
| `observed_ts` | Not needed for local-only; no relay/queue delay to measure |
| `resource` | Injected at read time from process metadata — saves ~100 bytes/event versus per-event embedding |
| `consent` | Not needed until remote sinks exist (v3) |
| `schema` | YAGNI until multiple envelope versions exist |
| `body` | Human summary derivable from `event + data` at display time; not worth storing |
| `seq` | Unnecessary when events are append-ordered within segments |

**What was renamed/simplified:**
- `correlation` → `ids` — shorter, same semantics, avoids OpenTelemetry context confusion.
- `scope` simplified from object to dotted string (e.g. `chat.server.ws`).

---

## Retention and Liveness

### D69: Spawn-store-based liveness for spawn segment retention {#d69}

*2026-05-02*

**Decided:** For segments whose `logical_owner` is a spawn ID (e.g. `p42`), retention uses `is_spawn_genuinely_active(runtime_root, spawn_id)` rather than raw process liveness (`os.kill(pid, 0)`). The function checks spawn store status (`queued/running/finalizing`) + heartbeat file freshness (< 120 s) + runner PID liveness. `cli` and `chat` segments continue to use raw process liveness.

**Why not raw PID for spawns:** A spawn process exits at harness termination, but the spawn record may still show `running` during the finalizing window (report extraction, event drain). Raw PID check would orphan the segment immediately, potentially deleting events written during finalization. Spawn-store-based liveness treats the store as the authority on whether the spawn is genuinely done.

**Why raw PID is fine for `cli`/`chat`:** These processes have no equivalent "finalizing" window. When the PID disappears, the process is done.

**Implementation:** `is_spawn_genuinely_active()` is in `lib/state/liveness.py`. It is read-only — it never mutates spawn state.

---

### D70: `--global` uses point-in-time directory walk {#d70}

*2026-05-02*

**Decided:** `meridian telemetry tail/query/status --global` discovers telemetry by walking `~/.meridian/projects/*/telemetry/` at invocation time. It also includes `~/.meridian/telemetry/` (legacy user-home segments) if that directory exists. No subscription or registration.

**Why point-in-time:** Cross-project queries are an operational and debugging use case, not a hot path. Snapshot discovery is simpler and has no stale-registration problem. A project whose telemetry dir was created after the CLI invoked `--global` is not included in that run — acceptable for a debug tool.

**Legacy segments:** `~/.meridian/telemetry/` is the pre-D67 write path and now has no active writers. Including it in `--global` reads lets old segments remain visible until they age out (7-day default). No migration tool needed; the directory disappears naturally.

---

### D-tel-binary-mode: `tail_events()` reads telemetry segments in binary mode {#d-tel-binary-mode}

*2026-05, Windows compat audit (PR #184)*

**Decided:** `tail_events()` (live-tail reader in `lib/telemetry/reader.py`) opens
segment files in binary mode (`'rb'`) and decodes lines explicitly. Cursor positions
are byte offsets, not character offsets.

**Why:** On Windows, text-mode `seek()` and `tell()` are unreliable for multi-byte
UTF-8 characters. `tell()` in text mode may return values that cannot be safely passed
back to `seek()` for mid-stream resumption — on Windows with `CRLF` translation or
multi-byte encodings the cursor can land in the middle of a character, producing a
`UnicodeDecodeError` on the next read. Binary mode bypasses all text-mode translation
and gives stable, portable byte offsets for cursor persistence.

**Why not Windows-only fix:** The binary mode path is simpler and equally correct on
POSIX. A conditional branch would add complexity for no benefit.

**Format contract:** Each line is a UTF-8-encoded JSON object terminated by `\n`.
Binary mode reads the raw bytes; the reader decodes each line with
`line.decode('utf-8').strip()` before JSON parsing. Lines that fail decode or JSON
parse are skipped and counted as malformed.
