# Decisions: Harness Connection Seams

The decisions here are harness connection seams that outlived the removed chat
backend: connection ownership,
the HITL handler protocol, the managed-primary passthrough boundary, and the
spawn depth cap. The protocol, pipeline, acquisition, command-layer and
final-gate records went with the code. The harness-semantics decision this page
used to hold (D20, separate semantic functions) was superseded by the single
`EventSemantics` descriptor; see [state decisions](state.md).

---

## Connection Ownership

### D15: SpawnManager is the sole consumer of a connection's event stream {#d15}

*2026-04-29, revised 2026-09-25 (PR 2, PR 3)*

**Decided:** `SpawnManager` (`lib/streaming/spawn_manager.py`) owns the
`HarnessConnection` lifecycle, the drain loop and cleanup. Nothing
else iterates `HarnessConnection.events()`. Delivery to other parties goes
through `SpawnManager._run_event_hooks` (inline hooks before subscriber
fan-out), not through a second reader. See
[attempt facts and delivery](../architecture/attempt-facts-and-delivery.md#the-emit-path).

**Why:** Claude's connection enforces single `events()` consumption, and the
drain loop is already that consumer. A second consumer of the same stream is not
possible.

**Rejected:** An adapter wrapping the connection directly (dual owner on a
single-consumer stream); bypassing `SpawnManager` (loses completion and cancel
integration); refactoring `HarnessConnection` into a `HarnessAdapter`
(invasive, high regression risk). The R4 observer seam
(`EventObserverRegistry`) was built for this and deleted in PR 2 once it had no
users.

---

### D16: HITL policy injection via handler protocol, not boolean flag {#d16}

*2026-04-29*

**Decided:** `ServerRequestHandler` (`lib/harness/connections/base.py`) is an
injectable policy object. `AutoAcceptHandler` is the default on the spawn path
and preserves existing behavior; `InteractiveHandler` routes approval requests
to a human. Handler capabilities such as `no_runtime_hitl` are declared on the
protocol and read directly (cleanup phase 1, B6).

**Why:** A boolean `auto_respond_requests` flag couples surface policy (who
decides) to transport code (how to send). A handler protocol separates the two
axes, so a new HITL behavior is a new handler, not a new flag and branch. The
seam was built first because it was safety-critical: Codex had been
auto-accepting every approval request and returning empty answers to user-input
requests, with no human visibility.

**Constraint:** Claude and OpenCode support approval only at launch time.
`HarnessConnection.respond_request()` raises `NotImplementedError` for them, and
`ConnectionCapabilities.supports_runtime_hitl` reports it.

---

### D23: Managed-primary passthrough is separate from the event consumer layer {#d23}

*2026-04-29, restated after the chat backend was removed*

**Decided:** `lib/harness/passthrough/` only builds the two artifacts a
managed-primary session needs: the `ConnectionConfig` for Meridian's own
connection and the attach command for the user's TUI. TUI input goes straight to
the harness backend; it never passes through Meridian's command routing. The
package is reached through `get_passthrough(harness_id)`.

**Why:** The TUI process has its own input channel to the backend, so routing
its input through Meridian adds latency and complexity for nothing.

**Claude-specific constraint:** Claude Code has no backend separation and no
`--attach`, daemon or local-socket mode. `ClaudePassthrough` always raises
`PassthroughError`; Claude primaries use the black-box PTY path.

---

## Nesting

### D32: Nesting is bounded by spawn `max_depth`, not per-command guards {#d32}

*2026-05-07*

**Decided:** Recursion is capped at the spawn layer by `[defaults] max_depth`
(`MERIDIAN_MAX_DEPTH`; built-in default 3, declared in `lib/config/models.py`),
enforced in `lib/ops/spawn/api.py` through `lib/core/depth.py`. At depth ≥ `max_depth`, further nested spawns are refused
regardless of which command asked. This replaced a nested-launch guard that
blocked `meridian chat` whenever `MERIDIAN_DEPTH > 0`.

**Why the guard was a problem:** Delegated smoke and browser testers run at
`MERIDIAN_DEPTH > 0`, so the guard silently blocked them from launching the very
process they were meant to test. A per-command guard was the wrong scope; a
depth cap bounds every recursion path at once.

**Evidence:** Smoke p4982, with the project set to `max_depth = 4`: a depth-3
spawn is allowed and a depth-4 spawn rejected.

**Commits:** fdad9656 (launch guard), 09c4dc23 (management subcommands),
8376b446 (changelog).
