# Chat Backend Extensibility

The chat backend is extended at nine defined points. Each point follows the same structure: a typed protocol the extension must implement, a registry or injection site where it registers, and a zero-touch guarantee listing what must not be modified.

## The Registry Pattern

All harness-layer extension points use the same structure:

1. **Protocol** — typed interface defining what an extension must implement
2. **Registry dict** — maps `harness_id` (or strategy name) to the implementation class
3. **Zero-touch guarantee** — everything outside the protocol and registry entry is unchanged

Three registries govern the harness layer:

```python
# src/meridian/lib/harness/connections/registry.py
CONNECTION_REGISTRY: dict[str, type[HarnessConnection]] = {
    "claude":    ClaudeCliConnection,
    "codex":     CodexAppServerConnection,
    "opencode":  OpenCodeHttpConnection,
}

# src/meridian/lib/chat/normalization/registry.py
NORMALIZER_REGISTRY: dict[str, type[EventNormalizer]] = {
    "claude":    ClaudeNormalizer,
    "codex":     CodexNormalizer,
    "opencode":  OpenCodeNormalizer,
}

# src/meridian/lib/harness/passthrough/registry.py
PASSTHROUGH_REGISTRY: dict[str, type[TuiPassthrough]] = {
    "claude":    ClaudePassthrough,      # rejection stub — no managed attach
    "codex":     CodexTuiPassthrough,
    "opencode":  OpenCodeTuiPassthrough,
}
```

The command handler's `match command.type:` applies the same pattern without a dict — one new match case registers the command.

## S1: New Harness

**Write:**

- `src/meridian/lib/harness/connections/<newharn>.py` — `NewHarnConnection` implementing `HarnessConnection` protocol (~200–400 lines). Parses wire events, calls `respond_request()` via injected `ServerRequestHandler`.
- `src/meridian/lib/chat/normalization/<newharn>.py` — `NewHarnNormalizer` implementing `EventNormalizer` protocol (~200–400 lines). Translates `HarnessEvent` → `list[ChatEvent]`. Import helpers from `common.py` and `synthetic.py`.

**Register:**

```python
CONNECTION_REGISTRY["newharn"] = NewHarnConnection    # connections/registry.py
NORMALIZER_REGISTRY["newharn"] = NewHarnNormalizer    # normalization/registry.py
```

**Do not touch:** `chat/server.py`, `chat/runtime.py`, `chat/session_service.py`, `chat/event_pipeline.py`, `chat/command_handler.py`, `streaming/spawn_manager.py`, any existing connection or normalizer files.

**Tests:** Normalizer unit tests with sample event sequences. Smoke: `meridian chat --harness newharn` produces `ChatEvent` output.

---

## S2: SDK Upgrade (CLI → SDK for an existing harness)

**Write:**

- `src/meridian/lib/harness/connections/claude_sdk.py` — `ClaudeSdkConnection` consuming typed SDK events instead of NDJSON. Output: same `HarnessEvent` types.

**Register:**

```python
CONNECTION_REGISTRY["claude"] = ClaudeSdkConnection   # one-line swap
```

**Do not touch:** `ClaudeNormalizer` (operates on `HarnessEvent`, not wire data), chat server, session service, event pipeline, command handler, all other connections and normalizers.

---

## S3: New Backend Acquisition Strategy

**Write:**

- New class implementing `BackendAcquisition` protocol (~100–200 lines). Examples: `WarmPoolAcquisition`, `ResumeAcquisition`, `RemoteAcquisition`.

**Inject:** Pass the new instance to `ChatSessionService.__init__` at server startup via `chat_cmd.py`. No registry file — injection point is server configuration.

**Contract:** `async acquire(chat_id: str, initial_prompt: str) -> BackendHandle`

**Do not touch:** `ChatSessionService`, event pipeline, normalizers, connections, command handler, REST endpoints, WebSocket handler, replay, checkpoint.

---

## S4: New Event Type (within an existing family)

**Write:**

- String constant in `src/meridian/lib/chat/protocol.py`.
- Emit point: one return path in the relevant harness normalizer.
- SQLite projection handler in `event_index.py` only if fast query access is needed.

**Register:** Nothing. New event types flow through `ChatEventPipeline.ingest()` transparently — the pipeline is type-agnostic.

**Do not touch:** `ChatEventPipeline`, `WebSocketFanOut`, `ReplayService`, chat server, session service, command handler.

---

## S5: New Event Family (e.g., `extension.*`)

**Write:**

- Type string constants, e.g. `"extension.microct.reconstruction_progress"`.
- Emit point: server-side service or harness normalizer.
- SQLite projection handler if queries are needed.

**Register:** Nothing. Unrecognized types pass through the pipeline unchanged.

**Namespace contract:** Use `extension.<domain>.` for domain-specific events (e.g., `extension.microct.*`), `extension.<harness>.` for harness-specific events (e.g., `extension.claude.thinking_budget`).

**Do not touch:** `ChatEventPipeline`, WebSocket handler, replay, chat server, session service, command handler, any existing normalizers.

---

## S6: New Command

**Write:**

- One match case in `ChatCommandHandler.dispatch()` (~5–10 lines)
- Handler function (~10–40 lines) using `command_invariants.py` guards
- Optional REST wrapper: construct `ChatCommand`, call `dispatch()`

**Register:** Nothing. The match case is the registration.

**Required:** Every new command must have a rejection path — `CommandResult(status="rejected", error=<reason>)` must be reachable.

**What the framework provides for free:** WebSocket transport, command correlation by `command_id`, centralized exception-to-rejection mapping, the invariant library, REST fallback, multi-client serialization via session lock.

**Do not touch:** `ChatEvent` protocol, event pipeline, WebSocket transport, REST routing framework, normalizers, connections, `SpawnManager`.

---

## S7: New Consumer (beyond browser WebSocket)

**Write:**

- Client that constructs `ChatCommand` objects and calls `ChatCommandHandler.dispatch()`.
- Reads from event log or subscribes via WebSocket to receive `ChatEvent` output.

**Register:** Nothing.

**Do not touch:** `ChatCommandHandler`, session service, event pipeline, WebSocket handler, REST endpoints, normalizers, connections.

---

## S8: New Observer (beyond JSONL + SQLite)

**Write:**

- New class implementing `EventObserver` protocol:
  - `async on_event(spawn_id: str, event: HarnessEvent) -> None`
  - `async on_complete(spawn_id: str) -> None`
  
  ~100 lines. Examples: metrics sink, streaming transcript writer.

**Register:**

```python
spawn_manager.register_observer(spawn_id, MyObserver(...))
```

**Contract:** Both methods must be async. `on_event` must not block — the `QueuedObserver` wrapper runs it asynchronously, but slow processing risks queue drop.

**Do not touch:** `ChatEventLog` (JSONL stays as source of truth), `ChatEventPipeline`, `ChatEventIndex`, `WebSocketFanOut`, replay.

---

## S9: New HITL Policy

**Write:**

- New class implementing `ServerRequestHandler` protocol (~50 lines). Example: `ConditionalHandler` (auto-approve read-only operations, escalate file writes).

**Inject:** Pass the instance to `HarnessConnection` constructor for harnesses supporting runtime HITL. Set in `ColdSpawnAcquisition` at connection build time.

**Contract:** `async handle_request(connection, request) -> None`. Must call `connection.respond_request()` or `connection.respond_user_input()` before returning — the harness transport blocks until a response arrives.

**Do not touch:** Transport methods on `HarnessConnection` (`respond_request`, `respond_user_input`), `BackendHandle`, chat server, command handler, normalizers.

---

## Why Each Boundary Holds

| Boundary | Decision | Mechanism |
|---|---|---|
| Wire parsing never leaves the connection | D8-superseded | Connections emit `HarnessEvent`; chat layer owns `HarnessEvent → ChatEvent` projection |
| Single `events()` consumer | D15 | Claude enforces one reader. SpawnManager is that reader. All others observe via `EventObserverRegistry`. |
| New events need no pipeline change | D18 | `ChatEventPipeline.ingest()` is type-agnostic; new types pass through unchanged |
| Separate semantic functions | D20 | `terminal_outcome`, `activity_transition`, `clears_signal` are independent functions; no combined struct |
| Single command dispatch | D21 | All commands enter `dispatch()`; new command = new match case, no new routing code |
| Acquisition strategy is injected | D19 | `ChatSessionService` accepts `BackendAcquisition` protocol; strategy change = constructor argument |
| Deferred commands use capability gating | D24 | `ConnectionCapabilities.supports_*` gates commands; flip the flag when harness gains capability |

## Extension Checklist

Before submitting any extension:

1. **No `lib/chat/` imports in `lib/harness/`** — harness layer must not depend on chat types. Normalizers live in `lib/chat/normalization/` precisely to keep `ChatEvent` imports inside the chat layer.
2. **No `lib/harness/` imports in `lib/streaming/`** — `SpawnManager` core must not acquire harness-specific knowledge.
3. **`extension.*` namespace for domain events** — or a canonical family name if the type is general enough for all harnesses.
4. **Registry entries are the only edits to existing files** — editing non-registry code in an existing harness file means the boundary is wrong.
5. **Every new command has a rejection path** — `CommandResult(status="rejected", error=<reason>)` must be reachable.

## Key References

- `src/meridian/lib/harness/connections/registry.py` — `CONNECTION_REGISTRY`
- `src/meridian/lib/chat/normalization/registry.py` — `NORMALIZER_REGISTRY`
- `src/meridian/lib/harness/passthrough/registry.py` — `PASSTHROUGH_REGISTRY`
- `src/meridian/lib/chat/commands.py` — `ChatCommand`, `CommandResult`
- `src/meridian/lib/chat/command_handler.py` — `ChatCommandHandler`
- `src/meridian/lib/chat/backend_acquisition.py` — `BackendAcquisition` protocol
- `src/meridian/lib/streaming/event_observers.py` — `EventObserver`, `QueuedObserver`, `EventObserverRegistry`
- `src/meridian/lib/chat/protocol.py` — `ChatEvent`, event family constants

## Related

- [normalization.md](normalization.md) — S1 normalizer details
- [command-system.md](command-system.md) — S6 command details
- [decisions/chat-backend.md](../../decisions/chat-backend.md) — D8, D15, D18, D19, D20, D21, D24
