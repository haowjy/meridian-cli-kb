# research/t3-code-reference — T3 Code Architecture Analysis

T3 Code was studied deeply during the chat backend design (2026-04-29). It is the most
complete prior art for a multi-provider coding agent protocol with browser UI. This doc
captures what was found, what was taken, and what was skipped.

---

## What T3 Code Is

A multi-provider coding agent platform with a React browser UI. Supports Claude, Codex,
OpenCode, and Cursor. Has a formal protocol between the harness layer and the frontend.

**Why it matters for meridian:** T3 Code is the primary existence proof that the
event taxonomy we chose works across diverse harnesses. Their normalization pipeline
is the most evolved we found.

---

## T3 Code's Architecture

### 4-Layer Normalization

```
Raw provider output (Claude NDJSON, Codex JSON-RPC, OpenCode SSE)
    ↓
ProviderRuntimeEvent (normalized, per-provider)
    ↓
OrchestrationCommand (client commands + internal engine commands)
    ↓
OrchestrationEvent (produced by decider, stored in event store)
    ↓
Read Model (projections → browser push)
```

Compare to meridian's simpler pipeline: `HarnessEvent` → normalizer → `ChatEvent` → persist + broadcast. Meridian skips the CQRS event store and read-model projection layers because we have one chat, not multi-project multi-thread orchestration.

### 25+ Command Types, Single Dispatch

T3 Code has 25+ command types in two tiers:
- **Client commands** (browser/CLI dispatched): `thread.turn.start`, `thread.turn.interrupt`, `thread.approval.respond`, `thread.checkpoint.revert`, `thread.session.stop`, etc.
- **Internal commands** (ingestion engine): `thread.message.assistant.delta`, `thread.turn.diff.complete`, etc.

All client commands route through a single `orchestration.dispatchCommand` WebSocket RPC method. The `decider.ts` is a pure function: `(command, readModel) → OrchestrationEvent[]`. This is the pattern we took for `ChatCommandHandler.dispatch()`.

### Command Correlation

Every command carries a `commandId` (UUID). Every produced event carries `correlationId`. Response to the browser is a `DispatchResult` with the assigned sequence number — not the actual effects. Effects arrive via the subscription stream. We took the correlation pattern; we skipped the CQRS/event-store separation.

### Centralized Invariant Validation

`commandInvariants.ts` provides reusable guards: `requireThread`, `requireProject`, `requireThreadNotArchived`, etc. We took this pattern as `command_invariants.py`.

---

## What We Took

| T3 Code feature | Meridian equivalent | Notes |
|---|---|---|
| Single `dispatchCommand` | `ChatCommandHandler.dispatch()` | Simpler — no event store, direct service calls |
| `commandId` for correlation | `command_id` in `ChatCommand` | Flows through to `CommandResult.ack` |
| Centralized invariant validation | `command_invariants.py` | `require_active_execution`, `require_state` |
| `ProviderRuntimeEvent` taxonomy | `HarnessEvent` + event families | Smaller (~30 vs ~50 types); no project/auth/audio |
| Turn/item/content/request event families | Same families in `ChatEvent` | Direct inheritance |
| Client/internal command split concept | Dropped | We have no internal commands — normalizer produces `ChatEvent` directly |

---

## What We Skipped

| T3 Code feature | Why skipped |
|---|---|
| Full CQRS — event store + projector + read model | Overkill for single-chat. T3 Code needs this for multi-project multi-thread orchestration + 26 migrations. We need one chat. |
| Project/workspace management commands | No multi-project scope in meridian chat |
| Internal command tier | Our normalizer produces `ChatEvent` directly — no intermediate orchestration event layer |
| Command sequence batching | No multi-command transactions needed |
| Account/auth events | Handled outside the chat protocol |
| Realtime audio events | Out of scope |
| MCP server events | Not needed for v1 |
| Checkpoint events (in T3) | We added our own `checkpoint.created` / `checkpoint.reverted` |

---

## AG-UI, ACP, Vercel AI SDK Analysis

Three other protocols were evaluated before designing the custom `ChatEvent` protocol (D1):

| Protocol | Why rejected |
|---|---|
| **AG-UI** | Lacks first-class events for file changes, checkpoints, and HITL approval. HITL spec is still draft. We'd use `CUSTOM` for the most important features — defeating the standard's purpose. |
| **ACP** | Control protocol for IDE↔agent communication over stdio. T3 Code doesn't expose ACP to the frontend — they normalize it. Using it as a frontend protocol is a category mistake. |
| **Vercel AI SDK** | Tightly coupled to TypeScript hooks. No session management, no HITL, no code operations. No interop benefit since we build the frontend ourselves. |

**Takeaway:** All three lack coverage for the most important events in our use case (HITL, file tracking, checkpoints). Building our own custom protocol was the right call.

---

## Other Prior Art

| Tool | Finding |
|---|---|
| **Aider** | Streamlit UI, no formal protocol. Not applicable. |
| **OpenCode** | No web UI. Connection is HTTP SSE (studied for `OpenCodeHttpConnection`). |
| **Cursor** | Closed source, browser extension model. Not studied in depth. |

---

## Cross-References

- [decisions.md](../decisions.md) — D1 (custom protocol rationale), D2 (event pipeline over CQRS), D5 (T3-inspired taxonomy), D18 (three-layer pipeline)
