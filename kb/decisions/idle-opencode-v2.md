# Decision: OpenCode 2.x Idle Sensor

This page records why OpenCode 2.x needs a separate idle sensor and how its
transport choice and long-lived primary observer were decided. The current
event, inbox, and reconnecting-stream mechanism lives in
[Idle Adapter Architecture](../architecture/idle-adapters.md).

Meridian-initiated compaction on both dialects (the `/api/session/{id}/compact`
and `/session/{id}/summarize` calls, the compaction waiter and its durable
completion check) was deleted with
[D-idle-compaction-removed](idle-notifications.md#d-idle-compaction-removed--meridian-no-longer-compacts-idle-sessions). The compaction parts of this record are
history. The V2 sensor still ignores inbox items of type `compaction`, and V1
still classifies OpenCode's own compaction messages, so a native compaction is
not taken for a user return.

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

**Decision:** Add a second sensor beside the frozen V1 sensor. The V2 sensor
uses execution and typed inbox events, and its compaction route is
`/api/session/{id}/compact`; V1 remains unchanged. Select the sensor from
`OpenCodeConnection.resolved_version`, the version chosen by the transport,
with the environment-based resolver only as fallback. A typed V2 user inbox
item is a return when enqueued, and delivery is also a return outside
Meridian's own compaction. The full event, inbox, and compaction contract is
in [Adapter contracts](../architecture/idle-adapters.md#adapter-contracts).

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
session got at most one idle stretch. The decision is that stream lifetime
follows the consumer: spawns retain stall → reconcile → terminal → end, while a
managed primary reconnects for its lifetime and never synthesizes a terminal.
The mode comes from the existing `_primary_observer_mode`; review rejected a
new `EventStreamLifetime` enum as a second name for the same fact. A
compaction missed during a reconnect is confirmed from durable data. The check
requires the compaction message to be `completed`, not just present, because
OpenCode writes that message at admission. The reconnect rules are in
[Adapter contracts](../architecture/idle-adapters.md#adapter-contracts); the
durable-data check went with Meridian's compaction.

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

- [Idle Notifications](idle-notifications.md): shared idle policy, config, and harness-hosting rationale.
- [Idle Adapter Architecture](../architecture/idle-adapters.md): current OpenCode V1/V2 contracts and known fragility.
- [D-idle-running-work-reporters](idle-notifications.md#d-idle-running-work-reporters--only-work-that-wakes-the-session-holds-the-push): why the V2 sensor reports background `shell` and `subagent` tool work (v0.10.30) and V1 reports none.
- [Idle TTL Detection](idle-ttl-detection.md): provider-derived TTL mapping.
- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md): provider retention evidence.
