# architecture/idle-adapters — Idle Adapter Architecture

> **Status:** Permission-prompt forwarding below is implemented on the
> unmerged `feat/permission-prompt-ping` branch for #684, based on v0.10.28.
> That branch names the event `session.needs_permission`; under the renamed
> hook contract (PR #688) it becomes `notify.needs_permission` when the branch
> rebases.

This page owns the current idle adapter mechanism: where it lives, what each
harness schedules, the runtime contracts, and the known fragility. The decision
record retains the rationale, rejected alternatives, and decision history in
[Idle Notifications](../decisions/idle-notifications.md).

## Where it lives

| Concern | Code |
|---|---|
| Notify delivery and CLI | `src/meridian/lib/notify/` (`emit.py` fires a notice as its hook event), `src/meridian/lib/hooks/builtin/notify.py` (the built-in `notify` route), `src/meridian/cli/notify_cmd.py`; see [D-notify-hook-events](../decisions/idle-notify-channels.md#d-notify-hook-events--notifications-are-hook-events-delivered-by-routes) |
| Idle policy, sidecar, CLI, and state | `src/meridian/lib/idle/` (`timeline.py` schedule, `guards.py` fire checks, `wake.py` running-work reading, `service.py` transitions, `sidecar.py` launcher loop), `src/meridian/cli/idle_cmd.py`, `src/meridian/lib/state/idle_store.py`; TTL detection reads the chat's `native_store` through `src/meridian/lib/state/session_store.py` |
| Harness ports and Python adapters | `src/meridian/lib/harness/idle_types.py`, `bundle.py`, `claude_idle.py`, `pi_idle.py`, `codex_idle.py`, `opencode_idle.py` (V1 and V2 sensors, provider TTL); OpenCode dialect dispatch reads `connections/opencode_connection.py` `resolved_version` |
| In-harness adapters | `src/meridian/claude_runtime/meridian-idle/`, `src/meridian/pi_runtime/extensions/meridian-idle/` |
| Launch hosting | `src/meridian/lib/launch/process/primary_attach.py` |
| Layering guard | `tests/unit/harness/test_layering.py` |
| Shell grammar shared with the CLI | `src/meridian/lib/cli_args.py` (`TOP_LEVEL_VALUE_FLAGS`, also read by `cli/bootstrap.py`) |
| Agent notification rule | meridian-base `skills/notify/` |

## Per-harness coverage

The idle timeline has two stages: the "waiting on you" push and the cache-expiry
warning. Claude additionally forwards a stateless still-waiting permission
signal outside that timeline. Meridian does not compact idle sessions; each
harness compacts on its own
([D-idle-compaction-removed](../decisions/idle-notifications.md#d-idle-compaction-removed--meridian-no-longer-compacts-idle-sessions)).
What each harness schedules, where its TTL comes from, and the last time the
path was seen working outside unit tests. "Live" means a real interactive
primary; "replay" means the installed CLI driven with recorded session facts
and scratch state.

| Harness | Arms on | TTL source | Stages | Last verified |
|---|---|---|---|---|
| Claude | mod: main-loop `turn.complete`, with the background tasks from the `classic.Stop` just before it | the transcript in the chat's recorded `native_store`: 3600 or 300, known | push, warn; the push is held or delayed by running work ([below](#running-work-and-the-push)) | TTL replay on installed 0.9.8, 2026-10-14: `ttl=3600` for a session that had changed directory. Running-work hold live on Claude Code 2.1.302 (`claude-haiku-4-5`), 2026-10-18: background `spawn wait` held then pushed after the wake; a `spawn list` loop only delayed; a subagent on a permission prompt held (`work:notify-hook-events`). On the unmerged #684 branch, the same Claude version emitted exactly one permission notice after an unanswered Bash prompt (`work:permission-ping`). |
| Pi | extension: `agent_end`, once `ctx.isIdle()` | provider and `PI_CACHE_RETENTION` table in `pi_idle` | push, warn on `anthropic`/`openai` long retention and `openai-codex`; push only otherwise | Replay on installed 0.9.8, 2026-10-14: `openai-codex` 1800 est. with the warning scheduled; `xai` push only. |
| Codex | `notify` callback on turn completion, pinned to the recorded thread | `[harness.codex.idle] ttl_seconds` default 1800, estimated | push, warn | Live on installed 0.9.8: the sidecar armed with the 1800 s schedule and a real turn sent the push (`work:idle-ttl-detection` D8; probes `spawn:p7759`, `spawn:p7767`). Reopen close confirmed live (`spawn:p7893`). |
| OpenCode 2.x | V2 sensor: `session.execution.{succeeded,failed,interrupted}` | provider read from `GET /api/session/{id}` at turn end; `openai` → 1800 est., everything else unknown | `openai`: push, warn. Others: push only | Released in v0.9.9 (PR #573); the stream fix shipped in v0.10.5. Live with opencode 2.0.28: `openai/gpt-6-luna` → push and warn with 1800 s offsets; other providers → push only; steer and queued prompts close the stretch. Released ≤0.9.8 arms nothing on 2.x. |
| OpenCode 1.x | frozen V1 sensor: `session.idle` | none (V1 never reports a provider) | push only | Probed live on 1.18.32 when the adapter was built (2026-10-08). Not re-run in phase 2; dispatch to it is covered by unit tests. |
| Cursor | — | — | none | No primary path. |

Only Claude reports running work today. Codex, OpenCode and Pi arms carry no
background tasks, so they get the ordinary push (#669).

Only Claude produces `session.needs_permission`. Its mod forwards
`classic.Notification` when, and only when, `notification_type` is
`permission_prompt`, through `meridian idle needs-permission`. Core emits the
notice through the ordinary notification hook routes without reading or
writing idle state. This signal neither releases the held waiting push nor
tracks the eventual answer. The decision and the rejected correlated designs
are in
[D-idle-permission-ping](../decisions/idle-notifications.md#d-idle-permission-ping--forward-claudes-still-waiting-signal-without-rebuilding-prompt-state).

### Hosts

Why each adapter runs where it does is in
[D-idle-adapter-hosting](../decisions/idle-notifications.md#d-idle-adapter-hosting--where-each-adapter-runs).

| Harness | Adapter host | How it gets there |
|---|---|---|
| Claude | mod inside Claude Code | bundled in meridian-cli, `--plugin-dir` on interactive launches only |
| Pi | bundled `meridian-idle` extension | `-e` entrypoint, interactive launch profile only |
| Codex | `notify` command plus a task in the attach launcher | `-c notify=[…]` on the app-server, interactive only |
| OpenCode | task in the attach launcher | bundle hook `primary_idle_sensor`; V1 or V2 sensor from the connection's resolved version |

The launcher task's lifecycle (started after the TUI, cancelled and awaited
before the connection stops) is in [Launch System](launch-system.md).

## Adapter contracts

Runtime probes (P0a–P0d) set these contracts, and G2 hardened them:

- **Claude.** A user return is `prompt.submit` or `command.run` with
  `origin.kind` `composer` or `bridge` (the phone/web client), not
  `turn.start`. Injected origins (a `task-notification`, a peer, a plugin) are
  not returns. Only the main loop's `turn.complete` (no `agentId`) arms.
  `classic.Notification` with `notification_type=permission_prompt` forwards
  Claude's message independently of the arm/return queue; every other
  notification type is ignored. The handler always calls `next(e)`, and its
  Meridian CLI task contains lookup and delivery failures rather than failing
  Claude's turn.
  Under `claude -p` (`isInteractive === false`, no surface) the mod never
  calls meridian.
- **Pi.** A user return is `input` with `source === "interactive"`. The handler
  cancels timers synchronously and queues `idle return` behind any in-flight
  arm without awaiting the CLI. Pi awaits input handlers, so an awaited CLI
  call cost every prompt about 0.5 s (G2 P1). `agent_end` arms on the next
  event-loop turn and only when `ctx.isIdle()`, passing `--provider` for TTL
  detection.
- **Codex.** `notify` fires on the app-server/TUI path and is pinned to
  meridian's recorded `harness_session_id`; threads with no stored state are
  dropped. `input-messages` is cumulative, so a return is a growth in its
  length, not a non-empty list; it reaches core as an arm with
  `--implies-return`. Codex has no return signal until a turn ends, so the
  sidecar asks the sensor's `busy()` before firing each stage and skips push
  and warn while the tmux pane shows `Working (` (G2 K2). Only sensors marked
  `external_events` (Codex) get that check.
- **OpenCode 2.x** (PR #573, v0.9.9; probed on 2.0.28). Events reach the
  sensor through the V2 transport's normalizer, which flattens `data` and keeps
  the envelope `id` as the turn ID. Only events for the launched parent
  `sessionID` count; child task sessions are ignored. The managed primary's
  `/api/event` stream stays open for the primary's lifetime and reconnects
  after quiet periods, EOF or recoverable errors (backoff capped at 10 s),
  never synthesizing a terminal (`connections/opencode_http.py`). The stream is
  live-only, so events during a reconnect are lost.
  - *Turn end* is any of `session.execution.succeeded`, `.failed` or
    `.interrupted`; `session.execution.started` marks busy. V2 emits neither
    `session.status` nor `session.idle`.
  - *User return* fires on `session.inbox.enqueued` for an item of type `user`,
    and again on `session.inbox.delivered` for that ID. Enqueue covers a prompt
    steered into the running execution, which never starts a new one. Delivery
    covers a queued prompt that runs after another execution has already armed
    a stretch. `session.inbox.cancelled` drops the ID. Inbox items of type
    `synthetic`, `compaction` and `move` never count as a return, so
    OpenCode's own compaction does not close the stretch.
  - Provider and message excerpts are fetched concurrently at turn end from
    `GET /api/session/{id}` (`data.model.providerID`) and
    `/api/session/{id}/message?limit=20&order=desc`. A failed provider read
    leaves the TTL unknown: push only.
- **OpenCode 1.x** (frozen). A user return is a new user message created after
  the last anchor, classified once `session.status` goes busy. OpenCode's own
  compaction (native `/compact` or autocompaction) posts a user message with a
  `compaction` part; the sensor records those message IDs and does not report
  them as returns. A turn end is `session.idle` after a busy status. Message
  excerpts come from `GET /session/{id}/message?limit=20`.

## Running work and the push

A turn can end while work it started is still running. Whether the push
should fire depends on whether that work will wake the session; the rationale
is
[D-idle-push-in-flight](../decisions/idle-notifications.md#d-idle-push-in-flight--the-push-waits-for-work-that-will-wake-the-session-the-warning-does-not).
The split is the usual one: the adapter reports facts in the host's terms,
and core alone interprets them.

**Claude reports it.** The only Claude Code surface that lists background work
is the `background_tasks` of the classic `Stop` hook, which fires just before
the main loop's `turn.complete`. The mod hooks it as `classic.Stop`, keeps the
list (cleared on `turn.start`), and forwards it on the arm as
`meridian idle arm --background-tasks=<json>`. Each entry is a kind plus, for a
shell, its command line: `subagent` and `workflow` → `agent`, `shell` →
`shell`, `monitor` → `watcher`, anything else → `other`. A turn with no `Stop`
forwards nothing and gets the ordinary push. `session_crons` are not forwarded.
When a task finishes, Claude Code delivers a `task-notification` prompt; that
is not a user origin, so it does not close the stretch, and its main-loop turn
arms again.

**Core reads it.** `BackgroundTask` (`lib/harness/idle_types.py`) is the wire
shape. `lib/idle/wake.py` reduces the list to one `ExpectedWake`:

| Running work | Reading | Effect in `timeline.schedule` |
|---|---|---|
| any `agent` | certain wake | no push this anchor; the response stays owed (`push_owed_since_ms`) until a push is sent, so the wake turn's push gets the long-turn delay |
| a `shell` whose last segment is foreground `meridian spawn` creation | certain wake | same as an agent |
| a `shell` whose last segment is `meridian spawn wait` | certain wake, plus a known wake time: `--timeout` (minutes), else `--yield-after-secs`, else the harness's wait-yield default | the warning is dropped if that time lands before the cache expires |
| any other `shell` (including `spawn --bg`, `spawn list`, loops, `spawn wait … && more`) or `other` | maybe | the push waits at least `quick_turn_push_seconds` |
| `watcher` | ignored | none |

`wake.py` lexes the line the way the shell would before handing it to
`shlex`. Newlines, `;`, `&&`, `||`, `|`, `&` and parentheses end a segment.
Redirections stay with their command. `#` comments are blanked, and a
backslash-newline joins the words around it. Only the **last** segment
decides, and a loop keyword in any segment makes the line *maybe*. The
command word is `meridian` (any path, after `VAR=value` assignments) or a
later `meridian` token behind `uv run` / `uvx`. Meridian's global options are
skipped using `lib/cli_args.py`. Spawn creation is recognized by an option
right after `spawn`, so a subcommand added later reads as *maybe*, never as
creation. Meridian's CLI grammar is parsed only here; an adapter must not
classify spawn commands itself. The rule and why it is shaped this way are in
[D-idle-push-in-flight](../decisions/idle-notifications.md#d-idle-push-in-flight--the-push-waits-for-work-that-will-wake-the-session-the-warning-does-not).
The arm reply carries `push_held` with the reason, which the Claude mod
writes to its log.

The Claude mod forwards at most 16 tasks and 512 characters per command, so a
large task list cannot hit the OS argument limit on `idle arm`. A cut command
ends in `\n;:`, which makes it read *maybe*, so a `--bg` lost past the cut
cannot make a detached spawn look like a certain wake.

## Close and reopen

Since v0.10.3 (PR #578): before exec, the primary launcher closes the resumed
session's open stretch with the `idle return` transform
([D-idle-reopen-return](../decisions/idle-notifications.md#d-idle-reopen-return--a-meridian-reopen-is-a-user-return)).
The call is `close_on_launch` in `lib/idle/service.py`, made from
`run_harness_process` in `lib/launch/process/runner.py` after
`prepare_prelaunch` and before the native observer and exec. Every interactive
primary passes it, for all four harnesses; managed-attach dispatch comes
later. No adapter changed. Closing a primary still never touches the store.

| Harness | On close | On start (reload or reopen) | After a Meridian reopen |
|---|---|---|---|
| Claude | `session.end` cancels in-memory timers only | `session.start` → `recover()` rebuilds timers from open rows in `idle status`, minus stages in `done`. Reload and resume take the same path; the mod can't tell them apart. | The row is already closed, so `recover()` re-adopts nothing. A reload runs no launcher, so its open row is rebuilt as before. |
| Pi | `session_shutdown` → `stop()` cancels timers only | `session_start` → `start()` → `idle status` → `restore()`, same as Claude | Same as Claude |
| Codex | launcher sidecar cancelled; timers die | never re-adopted: the new sidecar's identity-pin `arm` (`lib/idle/sidecar.py` `run`) re-anchors the row with a schedule from now, and the sidecar ignores that schedule until an external event moves the anchor | The launcher closes the old row first, so the pin opens a fresh stretch (`stretch + 1`, empty `done`). Its deadlines show in `idle status` but fire only after a real turn re-anchors. |
| OpenCode | launcher sidecar cancelled; timers die | never re-adopted: the sidecar polls the store only for external-event sensors | The old row is closed; the first turn end opens a fresh stretch |

No launcher runs for a Claude TUI started outside Meridian or for an in-TUI
`/resume`, so those starts can still rebuild an old open stretch.

## Known fragility

**Known fragility:** each adapter embeds harness behaviour that can change
upstream. Most drift fails closed: returns go unseen or a stage is skipped.
These cases fail open or fail silently:

- **Codex busy text `Working (`.** If it changes, `busy` is always false and
  the push or warning can fire while Codex is still working. This fails open,
  at the cost of a wrong notice.
- **Claude `classic.Stop` `background_tasks`.** If the hook stops firing
  before `turn.complete`, or the field is renamed, the mod forwards nothing
  and every turn gets the ordinary push while subagents or spawns run. If a
  task `type` is renamed, the mod maps it to `other` and the push is delayed
  rather than held. Both fail silently back to the pre-hold behaviour.
- **Claude's permission notification is an upstream dependency.** A subagent
  waiting on a permission prompt still holds the ordinary push because it is
  a running agent; `session.needs_permission` is the separate page. If Claude
  renames or stops emitting `classic.Notification` with
  `notification_type=permission_prompt`, Meridian silently stops paging for
  this case. Meridian intentionally does not recover by correlating
  `PermissionRequest` and `tool.call`
  ([D-idle-permission-ping](../decisions/idle-notifications.md#d-idle-permission-ping--forward-claudes-still-waiting-signal-without-rebuilding-prompt-state)).
- **The Claude mod fires a due push without asking whether the session is
  busy.** When finishing background work starts a new main-loop turn, the
  previous anchor's push timer can fire mid-turn. The launcher sidecar
  checks `busy()`; the mod does not (`lib/idle/.context/FUTURE`).
- **Return-signal renames** (Claude `origin.kind`, Pi `input.source`, Codex
  cumulative `input-messages`, OpenCode 2.x `session.inbox.*`). Stretches stop
  closing, so notifications stop after the first stretch. This fails silently.
- **Turn-end renames** (Claude `turn.complete`, Pi `agent_end`, OpenCode 2.x
  `session.execution.*`). The sensor never arms: no row, no push, no error.
  The #550 review filed this as "fails closed"; for idle, closed and silent are
  the same thing. OpenCode 2.x did exactly this for every release from 0.9.3 to
  0.9.8 ([lesson](../lessons/harness-integration.md#a-dialect-switch-must-move-every-consumer-of-the-stream)).
- **OpenCode 1.x `compaction` part type.** If OpenCode renames it, its own
  compaction message counts as a user return and closes the stretch, so the
  pending push and warning never fire. This fails silently.
- **Reopen close placement** (PR #578). The `close_on_launch` call must stay
  before exec in `run_harness_process`. Moved after the harness starts, Claude
  `recover()` and Pi `restore()` can re-adopt the old stretch first. The call
  also keys the row by `str(harness_adapter.id)` and reads the store under
  `child_env`'s user home; both must match what the harness's own
  `meridian idle` calls use (`claude`, `pi`, `codex`, `opencode`). A mismatch
  closes nothing: a reopened Claude or Pi primary re-adopts the old stretch and
  can push or warn on its old schedule, with no error. This fails silently.
- **TTL keyed by provider, not model** (Pi `openai-codex`, OpenCode `openai`).
  The 1800 s estimate assumes GPT-5.6 or later; an older model on those routes
  inherits it, so the warning can land after the cache is already cold. This
  fails silently; a configured `ttl_seconds` overrides it
  ([limitation](../decisions/idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd)).

## Related

- [Idle Notifications](../decisions/idle-notifications.md): rationale, rejected alternatives, and the adapter-hosting decision.
- [Idle TTL Detection](../decisions/idle-ttl-detection.md): launch-recorded transcript stores and provider TTL mapping consumed by this mechanism.
- [Harness Adapters](../codebase/harness-adapters.md#idle-bundle-ports): bundle ports and layering boundaries.
- [Harness integration lessons](../lessons/harness-integration.md): upgrade and dialect-drift failure modes.
- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md): provider retention facts behind the TTL defaults.
