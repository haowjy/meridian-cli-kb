# Decisions: Chat Backend

Decision records for the chat backend — why this architecture, what alternatives were rejected, what constraints drove the choices. Organized by subsystem.

See [../decisions.md](../decisions.md) for the chronological index. Architecture mechanism is documented in [../architecture/chat/](../architecture/chat/).

---

## Protocol and Pipeline Design

### D1: Custom ChatEvent protocol, not AG-UI / ACP / Vercel AI SDK {#d1}

*2026-04-29*

**Decided:** Build a custom `ChatEvent` protocol for the backend→frontend event stream.

**Why:** Three existing standards were evaluated:

- **AG-UI** — lacks first-class events for file changes, checkpoints, and HITL approval (HITL spec still draft at evaluation time). Any use would require `CUSTOM` events for the most important cases, defeating the standard.
- **ACP** — an IDE↔agent stdio control protocol. Using it as a frontend event stream is a category mistake; the abstraction level is wrong.
- **Vercel AI SDK** — no session management, no HITL, no code operations, TypeScript-only. The frontend is built in-house, so there is no interop benefit from a standard that doesn't fit.

All three would require `CUSTOM` for the highest-value events. Building custom from the start gives a coherent protocol rather than a patchwork.

**Evidence:** T3 Code (studied in depth) normalized 4 layers to their own custom event model, not AG-UI or ACP. See [../research/t3-code-reference.md](../research/t3-code-reference.md).

---

### D2: Event pipeline with replay, not thin normalization or full CQRS {#d2}

*2026-04-29*

**Decided:** Event pipeline architecture — per-chat JSONL event log, multi-client fan-out, replay from last sequence number.

**Alternatives compared:**

- **Option A: Thin normalization layer** — normalize events, emit to WebSocket. No persistence. On disconnect, all state is lost; reconnecting clients get nothing. Single-subscriber limit (one WebSocket consumer). Rejected: unacceptable for reconnect UX.
- **Option B: Event pipeline with replay** — chosen. ~1700 lines of new code. Gives reconnect, multi-client fan-out, and a queryable audit log.
- **Option C: Full CQRS/event sourcing** — SQLite event store, projector per read model, command bus, ~3000–4000 lines. Appropriate for T3 Code (multi-project multi-thread orchestration, 26 database migrations). Rejected: disproportionate for single-chat scope.

---

## Connection Ownership and Observation

### D15: SpawnManager owns the connection; chat pipeline observes via R4 seam {#d15}

*2026-04-29*

**Decided:** SpawnManager owns the HarnessConnection lifecycle, drain loop, `history.jsonl` persistence, heartbeat, and cleanup. The chat pipeline observes events through the R4 observer seam (`EventObserverRegistry`). No separate `HarnessAdapter` wrapper class.

**Why:** Claude's `HarnessConnection` enforces single `events()` consumption. SpawnManager already owns the drain loop — that is the sole consumer. Creating a second consumer of the same stream is not feasible. The "adapter" concept is realized by three collaborators:
- **SpawnManager** — owns connection and drain loop
- **ChatEventObserver** — normalizes HarnessEvent → ChatEvent
- **BackendHandle** — routes commands (inject, cancel, respond_request)

**Alternatives rejected:**
- Adapter wrapping the connection directly — creates a dual-owner conflict on the single-consumer stream.
- Bypassing SpawnManager entirely — loses heartbeat, completion, cancel integration.
- Refactoring HarnessConnection into a HarnessAdapter — high regression risk, invasive scope change.

---

### D16: HITL policy injection via handler protocol, not boolean flag {#d16}

*2026-04-29*

**Decided:** R5 adds `ServerRequestHandler` as an injectable policy object. `AutoAcceptHandler` is the default (spawn path, preserves existing behavior). `InteractiveHandler` is used by the chat backend (routes approval requests to the browser UI).

**Why:** A boolean `auto_respond_requests` flag couples surface policy (who decides) to transport code (how to send). A handler protocol separates the two axes. New HITL behaviors (conditional approval, role-based routing) require a new handler, not a new flag and branch.

**Constraint:** Claude and OpenCode do not support runtime approval — only launch-time. `HarnessConnection.respond_request()` raises `NotImplementedError` for these; `ConnectionCapabilities.supports_runtime_hitl` flags this at the capability level.

---

### D20: Separate semantic functions, not an umbrella classifier {#d20}

*2026-04-29*

**Decided:** R6 centralizes harness semantics as separate pure functions — `terminal_outcome`, `activity_transition`, `clears_signal` — in `harness/semantics.py`, rather than a single `classify_event()` returning a combined struct.

**Why:** Each consumer needs different semantic fields. A combined struct couples terminal policy, activity state, and connection bookkeeping. When any one concern changes, the combined struct widens the edit surface. Separate functions with shared helpers keep each consumer's dependency narrow.

---

## Normalizer Architecture

### D8: Normalizers in harness layer [SUPERSEDED] {#d8}

*2026-04-29 — superseded 2026-04-30 by D8-superseded*

**Original decision:** Per-harness normalizers live under `lib/harness/normalizers/`, not `lib/chat/normalizers/`.

**Original rationale:** Harness-specific event interpretation was already scattered across terminal classification and primary attach. Putting normalizers in the chat layer would push harness knowledge further upward.

**Why this was wrong:** Normalizers import `ChatEvent` from `chat/protocol.py`. The harness layer was already depending on chat types. The projection of `HarnessEvent → ChatEvent` is a chat concern, not a harness concern. Locality to raw event shapes was optimized for proximity at the cost of a backwards import direction.

---

### D8-superseded: Normalizers moved to chat/normalization/ {#d8-superseded}

*2026-04-30*

**Decided:** Delete `harness/normalizers/`. Move `ClaudeNormalizer`, `CodexNormalizer`, `OpenCodeNormalizer` to `chat/normalization/`. Registry and protocol moved intact. Shared helpers in `common.py`, synthetic event adapter in `synthetic.py`.

**Why the import direction was backwards:** Normalizers imported `ChatEvent` from `chat/protocol.py`, making the harness layer depend on chat types. The projection is a chat concern. The refactor makes import direction correct: harness layer stops at raw events + runtime semantics; chat layer owns the full HarnessEvent → ChatEvent projection.

**Three semantic planes that must stay separate:**

| Plane | Module | What it covers |
|---|---|---|
| Runtime/drain semantics | `harness/semantics.py` | `terminal_outcome`, `activity_transition`, `clears_signal` |
| Synthetic runtime events | `chat/normalization/synthetic.py` | Adapter for `meridian/*` event constants |
| Projection helpers | `chat/normalization/common.py` | Shared coercion helpers |

Runtime semantics (when does a drain end?) and projection semantics (which `ChatEvent` to emit?) are different questions. Merging them couples unrelated consumers.

---

## Acquisition

### D10: Pre-implementation refactors required before chat feature work {#d10}

*2026-04-29*

**Decided:** Three refactors complete before chat implementation begins:
1. **R5** — HITL seam on `HarnessConnection`. Safety-critical: Codex was auto-accepting all approval requests and returning empty answers to user input. Non-technical users need HITL as primary safety mechanism.
2. **R4** — Observer seam from SpawnManager (`EventObserverRegistry`, `QueuedObserver`). Required for reliable event delivery without dual-consumer conflict.
3. **R6** — Centralize harness semantic interpretation into `harness/semantics.py`.

**Why refactor first:** Building chat on weak seams would compound structural debt. R5 in particular was dangerous — the auto-answer behavior silently approved file writes and command execution with no human visibility.

**Alternative rejected:** Build chat first, refactor later — compounds coupling and requires a second disruptive refactor pass.

---

### D19: Backend acquisition deferred to first prompt {#d19}

*2026-04-29*

**Decided:** Backend is acquired on first prompt, not on chat creation. `POST /chat` reserves a `chat_id`, creates an event log, and sets state to `idle`. `POST /chat/{id}/msg` triggers acquisition.

**Why:** All three current `HarnessConnection` implementations send the initial prompt during `start()`. There is no "start backend in idle" mode. Deferring avoids either a fake empty prompt (muddies turn semantics) or a refactor across all three connections (high risk, not worth it for v1).

**Future acquisition strategies** (warm pool, resume, remote) implement `BackendAcquisition` without changing `ChatSessionService` or transport.

---

## Command Layer

### D21: Single command handler, not per-endpoint logic {#d21}

*2026-04-29*

**Decided:** All inbound commands route through `ChatCommandHandler.dispatch()`. REST endpoints and WebSocket frames produce the same `ChatCommand` type. Adding a command = adding a type + a match case in one place.

**Why:** T3 Code's single `dispatchCommand` demonstrates this at scale (25+ command types, one dispatch point, centralized invariants). Per-endpoint handlers duplicate session lookup, state validation, and error handling. Every new command would require a new endpoint, request type, and handler — three edits instead of one.

**Alternative rejected:** Per-endpoint handlers — duplicated routing logic and no centralized invariant enforcement.

---

### D22: Bidirectional WebSocket with REST fallback {#d22}

*2026-04-29*

**Decided:** The WebSocket at `/ws/chat/{id}` supports both directions. Frame discrimination: `type` field = ChatEvent (outbound), `command_type` field = ChatCommand (inbound), `ack` field = CommandResult. REST endpoints remain as thin wrappers.

**Why:** Single-connection bidirectional transport reduces client complexity and latency for interactive HITL flows. REST serves simple integrations and `curl` testing.

**Alternatives rejected:**
- WS-only — breaks `curl` testing.
- REST-only — requires maintaining both WS and HTTP connections simultaneously; high latency for HITL approval flows.

---

### D24: Deferred commands recognized but explicitly rejected at dispatch {#d24}

*2026-04-29*

**Decided:** `swap_model` and `swap_effort` are in the schema but rejected at dispatch with `not_supported_by_current_harness`. No current harness supports runtime model switching.

**Why:** Including deferred commands in the schema lets the frontend distinguish "unknown command" from "unsupported command" — different error handling paths. When a harness adds the capability, the handler adds a case and checks `ConnectionCapabilities.supports_runtime_model_switch`.

**Alternatives rejected:**
- Omit from schema entirely — worse frontend UX.
- Implement as no-op — misleading; should fail explicitly.

---

### D23: TUI passthrough is separate from the chat backend {#d23}

*2026-04-29*

**Decided:** TUI passthrough serves `meridian launch`, not `meridian chat`. The chat backend replaces the TUI — the browser is the UI. Passthrough and chat share connection classes but not the consumer layer. TUI user input goes directly to the harness backend, not through `ChatCommand`.

**Why:** The TUI subprocess has its own input channel to the harness backend. Routing through `ChatCommand` adds latency and complexity for no benefit. The passthrough silo is independently deletable — removing it touches no chat backend code.

**Claude-specific constraint:** Claude Code is a monolithic CLI with no backend separation and no `--attach`, `--daemon`, or local socket mode. `harness/passthrough/claude.py` is a rejection stub for the registry; `meridian launch` with Claude uses the black-box PTY path directly.

---

## Recovery

### D27: Final gate fix — 4 blocking correctness issues {#d27}

*2026-04-29*

**Decided:** Four blocking correctness issues from final gate review were fixed before ship; four structural findings were deferred.

**Blocking fixes:**
1. **Execution generation mismatch** — pass session's current generation into `ColdSpawnAcquisition.acquire()`, not static at construction time. Root cause of draining wedge and multi-turn breakage.
2. **Harness selection not honored** — `ConnectionConfig.harness_id` must flow through to the actual connection factory.
3. **Checkpoint scope safety** — refuse checkpoint create/revert when multiple chats exist under the same project root.
4. **Acquisition failure cleanup** — wrap `start_spawn + start_heartbeat` in try/except with rollback.

**Deferred structural (non-blocking):** Normalizer boundary direction, `server.py` runtime extraction, event interpretation deduplication, speculative capability surface cleanup.

---

### D29: SpawnManager `_history_writers` leak accepted as pre-existing debt {#d29}

*2026-04-29*

**Decided:** `SpawnManager.start_spawn()` allocates `_history_writers[spawn_id]` before `control_server.start()`. If the control server fails, the history writer entry leaks. Accepted as pre-existing SpawnManager debt, not a chat backend introduction.

**Why:** Not introduced by chat backend work. `control_server.start()` failures are rare (Unix socket binding). The leaked entry is small and doesn't affect correctness. Should be addressed in a focused SpawnManager cleanup.

---

### D30: Structural refactor findings are pre-existing, not regressions {#d30}

*2026-04-30*

**Decided:** Concurrency and safety findings from the Phase 7–11 final gate review — event_log truncation race, fanout teardown leak, SQLite connection leak, checkpoint cross-process locking, stop/acquisition race — are pre-existing architectural concerns, not regressions from the refactor.

**Why:** The affected files (`event_log.py`, `ws_fanout.py`, `checkpoint.py`, `event_index.py`, `session_service.py`) were not modified in any of the 5 refactor commits. The refactors were explicitly behavior-preserving. Blocking a structural cleanup on pre-existing concerns would scope-creep it into a behavioral rewrite.

---

### D31: Post-refactor structural notes accepted as next-iteration cleanup {#d31}

*2026-04-30*

**Decided:** Three medium structural concerns identified during Phase 7–11 final gate are accepted as next-iteration cleanup, not blocking:
1. `ColdSpawnAcquisition` in `chat/` carries launch policy — may belong in its own module.
2. `runtime`/`recovery` hidden bootstrap cycle via lazy import in `start()`.
3. Harness chat policy scattered across 5 files.

Plus one low-severity: passthrough naming ambiguity.

**Why:** These are genuine improvements that the refactor made more visible (a sign the refactor worked), but they don't block the refactor's goals. The refactor reduced entropy; these are next-iteration cleanup.

---

## Nested Chat Launch

### D32: Nested chat launch/management guard removed; depth-cap moved to `max_depth` config {#d32}

*2026-05-07*

**Decided:** Remove the nested-launch guard that blocked `meridian chat` launch and management subcommands when running inside a spawn (`MERIDIAN_DEPTH > 0`). Replace per-command depth enforcement with a project-level `[defaults] max_depth = 4` in `meridian.toml`.

**Commits:** fdad9656 (launch guard), 09c4dc23 (management subcommands), 8376b446 (changelog).

**Why the guard existed:** The original guard prevented recursive `meridian chat` launches that could spiral — a chat spawning another chat spawning another chat with no bound. The mechanism was sound but the scope was too broad.

**Why the guard was a problem:** Delegated smoke testers and browser testers (e.g. `@browser-tester`, `@smoke-tester`) need to launch a real live `meridian chat` process E2E to verify chat-shared-policy bugs. With the guard in place, these agents ran at `MERIDIAN_DEPTH > 0` and were silently blocked from launching chat — the very behavior they were meant to test. Chat-specific bugs only visible in live harness runs were therefore invisible to automated testing.

**What changed:**
- Chat launch and chat management subcommands no longer check `MERIDIAN_DEPTH`.
- Depth enforcement moved to the spawn layer: `[defaults] max_depth = 4` in `meridian.toml` caps total spawn nesting. At depth ≥ 4, further nested spawns are blocked regardless of command.
- A nested chat instance can launch, serve requests, and be tested by agents running at any depth below the cap.

**Evidence:**
- Smoke p4982: `meridian config show` confirms `max_depth = 4`; depth=3 spawn allowed, depth=4 rejected.
- Live rerun p5012: nested `uv run meridian chat --headless --port 0 --harness codex -m codex --agent meridian-subagent --skills playwright-cli --approval auto` launched inside a spawn, API reached `/openapi.json`, `POST /chat` succeeded, prompt returned `OK`, and `chat.configured` showed `requested_model_token=codex`, `model=gpt-5.3-codex`.

**Accepted risk:** Nested chat instances share the same state root as the parent process. `ChatRuntime.recover_all()` at startup does not scope discovery to the spawning context — a nested chat instance may discover and attempt recovery of sessions that belong to the parent or another sibling instance. This is low-risk for short-lived smoke/test runs (where the nested process starts, runs, and exits before the parent restarts), but is a latent correctness hazard for longer-lived or concurrent nested chat scenarios.

**Deferred:** Scoped nested chat runtime roots — keying each chat instance's runtime root by spawn scope to prevent cross-talk in discovery, recovery, and session-state. See [open-questions/future-work.md](../open-questions/future-work.md#scoped-nested-chat-runtime-roots) and design-lead spawn p4971 for the recommended architecture.
