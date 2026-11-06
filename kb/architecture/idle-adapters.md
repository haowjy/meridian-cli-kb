# Idle adapter architecture

Harness adapters report idle and background-work facts; core owns the notification decision so policies do not drift among hosts. See the source checkout's `src/meridian/lib/idle/.context/CONTEXT.md` and [idle notifications](../decisions/idle-notifications.md).

## Per-harness coverage

**Known policy divergence:** the fixed 1800-second signed-in Codex cache warning
is stale under the owner-confirmed indefinite-retention contract. Its
replacement belongs to [#707](https://github.com/haowjy/meridian-cli/issues/707),
not the child-completion handshake. The TTL values below describe the existing
idle implementation, not a guarantee for signed-in Codex. See
[cache policy](../research/prompt-cache-retention.md) for authentication scope.

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
| Claude | mod: main-loop `turn.complete`, with the background tasks from the `classic.Stop` just before it | the transcript in the chat's recorded `native_store`: 3600 or 300, known | push, warn; the push is held or delayed by running work ([below](#running-work-and-the-push)) | TTL replay on installed 0.9.8, 2026-10-14: `ttl=3600` for a session that had changed directory. Running-work hold live on Claude Code 2.1.302 (`claude-haiku-4-5`), 2026-10-18: background `spawn wait` held then pushed after the wake; a `spawn list` loop only delayed; a subagent on a permission prompt held. The permission notice, on its branch before v0.11.0 with the same Claude version, was emitted exactly once after an unanswered Bash prompt. |
| Pi | extension: `agent_end`, once `ctx.isIdle()` | provider and `PI_CACHE_RETENTION` table in `pi_idle` | push, warn on `anthropic`/`openai` long retention and `openai-codex`; push only otherwise; the push is held or delayed by tracked background bash | Replay on installed 0.9.8, 2026-10-14: `openai-codex` 1800 est. with the warning scheduled; `xai` push only. Running work live on Pi 1.2.0 (`openai-codex/gpt-5.6-luna`), 2026-10-18, before v0.10.29: a finished background managed `sleep 20` woke Pi with no input; a plain background shell delayed the push to the quick-turn delay; a background `spawn wait` held it until the wake; a detached `spawn wait` was not reported. |
| Codex | `notify` callback on turn completion, pinned to the recorded thread | `[harness.codex.idle] ttl_seconds` default 1800, estimated | push, warn | Live on installed 0.9.8: the sidecar armed with the 1800 s schedule and a real turn sent the push. Reopen close confirmed live. |
| OpenCode 2.x | V2 sensor: `session.execution.{succeeded,failed,interrupted}` | provider read from `GET /api/session/{id}` at turn end; `openai` → 1800 est., everything else unknown | `openai`: push, warn. Others: push only. The push is held or delayed by background `shell` and `subagent` tool work | Released in v0.9.9 (PR #573); the stream fix shipped in v0.10.5. Live with opencode 2.0.28: `openai/gpt-6-luna` → push and warn with 1800 s offsets; other providers → push only; steer and queued prompts close the stretch. Released ≤0.9.8 arms nothing on 2.x. Running work (v0.10.30): live on opencode 2.0.29 against an isolated `opencode serve`, background `shell` and `subagent` tool runs woke the parent with no user input, and the sensor held and re-armed the push across permission asks and a forced stream gap. |
| OpenCode 1.x | frozen V1 sensor: `session.idle` | none (V1 never reports a provider) | push only | Probed live on 1.18.32 when the adapter was built (2026-10-08). Not re-run in phase 2; dispatch to it is covered by unit tests. |
| Cursor | — | — | none | No primary path. |

Claude, Pi and OpenCode 2.x report the background work a turn left running, so
their push waits for work that will wake the session
([below](#running-work-and-the-push)). Codex and OpenCode 1.x report none and
get the ordinary push.

Only Claude produces `notify.needs_permission`. Its mod forwards
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

## Running work and the push

Only work that will wake the session can hold a push. Misclassifying uncertain work as certain can suppress a needed notification indefinitely. See [idle notifications](../decisions/idle-notifications.md#d-idle-push-in-flight--the-push-waits-for-work-that-will-wake-the-session-the-warning-does-not).

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
  a running agent; `notify.needs_permission` is the separate page. If Claude
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

