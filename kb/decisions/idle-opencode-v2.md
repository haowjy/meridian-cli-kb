# Decision: OpenCode 2.x Idle Sensor

This page records why OpenCode 2.x needs a separate idle sensor and how its
transport choice and long-lived primary observer were decided. The current
event, inbox, compaction, and reconnecting-stream mechanism lives in
[Idle Adapter Architecture](../architecture/idle-adapters.md).

## D-opencode-idle-v2 — OpenCode 2.x gets its own sensor, chosen by the resolved transport

**Status:** merged in PR #573, released in v0.9.9 (merge `04127891c`). Builds
up to 0.9.8 run only the V1 sensor; running OpenCode primaries keep the sensor
they started with until restarted. (`work:idle-ttl-detection` D6, with the
review-2 and review-3 fixes.)

**Problem it fixed:** none of 10 interactive OpenCode primaries on Oct 12–13
wrote any idle state, one of them three hours long. No sidecar error was
logged. OpenCode 2.0.28 ends a turn with `session.execution.succeeded`, and the
sensor waited for V1's `session.idle`, which 2.x never sends. Its compaction
route was also gone: `POST /session/{id}/summarize` returns 405 on 2.x and
`/api/session/{id}/summarize` returns 404.

**Decision:**

- **A second sensor beside the frozen V1 one.** `OpenCodeV2IdleSensor` reads
  V2 execution and inbox events and compacts through `/api/session/{id}/compact`
  ([adapter contract](../architecture/idle-adapters.md#adapter-contracts)). Both dialects share `_OpenCodeIdleSensorBase` for event
  plumbing, pane capture and requests. Dialect-only state lives in each
  subclass. V1 code is unchanged.
- **Dispatch on the version the connection resolved.** `primary_idle_sensor`
  reads `OpenCodeConnection.resolved_version`, set once when the dispatcher
  picks a transport at `start()`. Only if that is absent does it call
  `resolve_opencode_version` on `MERIDIAN_HARNESS_OPENCODE_VERSION`, the
  connection's own resolver. The sensor must speak the dialect the transport
  is emitting, so the transport's choice is the one source of truth.
- **The user return fires on enqueue, not at the next execution start.** V2
  inbox items are typed, so a `user` item is a positive identification the
  moment it is enqueued. Delivery fires it again (outside Meridian's own
  compaction) so a queued prompt that runs after an armed stretch still closes
  it.

**Rejected:**

- **Dispatching on the env var's value** (the first cut). Launch writes the
  *preference* into `MERIDIAN_HARNESS_OPENCODE_VERSION`, and the default is
  `auto`. Treating anything but `v1` as V2 handed a 1.x user on the default
  config the V2 sensor, which never fires on V1 events: a regression for the
  one dialect that worked. The dispatch test parametrized `None`, `v1` and
  `v2` but not `auto`, the value real launches carry (review-2 H1).
- **Re-probing `opencode --version` in the sensor** as the primary path: a
  second resolution can disagree with the transport. It stays only as the
  fallback.
- **Holding user inbox IDs until the next `session.execution.started`**, which
  is how V1 tells its own compaction message from a real prompt. On V2 the
  default delivery is `steer`: a prompt typed while the agent works runs at
  the next step boundary, inside the current execution, so no new start
  follows. The ID stayed pending. When Meridian's compaction started an
  execution later, the stale ID surfaced as a user return, which closed the
  stretch and re-armed a fresh push, warning and compaction on a session the
  user never came back to (review-2 M1). Firing only on enqueue then missed
  queued delivery: the queued prompt runs after an earlier terminal armed a
  stretch, so the push fired while the agent was busy (review-3 M1′). Hence
  enqueue plus delivery. Review 4 added two guards: no delivery-return while
  Meridian is compacting (a stranded steer delivered inside the compaction),
  and discard on `session.inbox.cancelled`.
- **Subclassing the V1 sensor.** The first cut inherited V1-only state that
  V2 never read, including a `_compacting` flag that gated nothing (review-2
  L4). Replaced by the shared base.
- **A compaction result taken from the POST response.** V2 acknowledges
  admission immediately; only the event sequence says the compaction finished.

**Observed:** OpenCode 2.0.28 ran a queued prompt inside the *same* execution
(one terminal), so the two-execution queue shape exists only in tests. The fix
covers both shapes.

**Stream lifetime (PR #583, v0.10.5; `work:idle-close-no-compact` D8).** As
released in v0.9.9, the sensor went deaf about two minutes after every turn.
The shared `events()` loop ended the stream after its 120 s liveness-stall
reconciliation. That is right for a spawn, which needs a terminal. For a
primary it meant Meridian's compaction completion was never seen (it timed out
as `failed` after 300 s) and later turns were never seen either. Each 2.x
session got at most one idle stretch. The decision:

- **The stream's lifetime follows its consumer.** Spawns keep
  stall → reconcile → terminal → end. A managed primary's observer keeps one
  stream for the primary's lifetime. When it goes quiet, hits EOF or a
  recoverable error, it reconnects with backoff capped at 10 s. It never
  synthesizes a terminal. The mode comes from the existing
  `_primary_observer_mode`. Review rejected a new `EventStreamLifetime` enum as
  a second name for the same fact.
- **Compaction is confirmed from durable data after a gap.** `/api/event` is
  live-only, so a reconnect can miss `session.compaction.ended`. If the waiter
  times out, the sensor accepts the compaction only when it finds the
  compaction message for the admitted inbox ID with `status == "completed"`
  and the session is idle with a successful outcome at or after admission. The
  message exists from admission onward, so its existence alone proves nothing.

Live check: a Meridian-initiated compaction came back `ok` with a native
compaction marker, the next turn opened stretch 2, and a one-shot OpenCode
spawn still finalized.

**Deferred** (`lib/harness/.context/FUTURE`): split the V2 sensor into
`opencode_idle_v2.py` when it grows again, and split `events()` into a
per-response drain with small lifetime policies for spawns and primaries.

**Codex needed no change** (`work:idle-ttl-detection` D8). No interactive Codex
primary had run since #550 shipped, so there was no evidence either way. A
live check on installed 0.9.8 showed the sidecar arming with the 1800 s
schedule and a real turn sending the push (see [Per-harness coverage](../architecture/idle-adapters.md#per-harness-coverage)).
Warn and compact share the core timeline with Claude, where they have fired,
so they were not waited out.

## Related

- [Idle Notifications and Idle Compaction](idle-notifications.md): shared idle policy, config, and harness-hosting rationale.
- [Idle Adapter Architecture](../architecture/idle-adapters.md): current OpenCode V1/V2 contracts and known fragility.
- [Idle TTL Detection](idle-ttl-detection.md): provider-derived TTL mapping.
- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md): provider retention evidence.
