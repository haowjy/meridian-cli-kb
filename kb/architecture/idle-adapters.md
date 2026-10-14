# architecture/idle-adapters — Idle Adapter Architecture

This page owns the current idle adapter mechanism: where it lives, what each
harness schedules, the runtime contracts, and the known fragility. The decision
record retains the rationale, rejected alternatives, and decision history in
[Idle Notifications](../decisions/idle-notifications.md).

## Where it lives

| Concern | Code |
|---|---|
| Notify delivery and CLI | `src/meridian/lib/notify/`, `src/meridian/cli/notify_cmd.py` |
| Idle policy, sidecar, CLI, and state | `src/meridian/lib/idle/`, `src/meridian/cli/idle_cmd.py`, `src/meridian/lib/state/idle_store.py`; TTL detection reads the chat's `native_store` through `src/meridian/lib/state/session_store.py` |
| Harness ports and Python adapters | `src/meridian/lib/harness/idle_types.py`, `bundle.py`, `claude_idle.py`, `pi_idle.py`, `codex_idle.py`, `opencode_idle.py` (V1 and V2 sensors, provider TTL); OpenCode dialect dispatch reads `connections/opencode_connection.py` `resolved_version` |
| In-harness adapters | `src/meridian/claude_runtime/meridian-idle/`, `src/meridian/pi_runtime/extensions/meridian-idle/` |
| Launch hosting | `src/meridian/lib/launch/process/primary_attach.py` |
| Layering guard | `tests/unit/harness/test_layering.py` |
| Agent notification rule | meridian-base `skills/work-artifacts/SKILL.md` |

## Per-harness coverage

What each harness schedules, where its TTL comes from, and the last time the
path was seen working outside unit tests. "Live" means a real interactive
primary; "replay" means the installed CLI driven with recorded session facts
and scratch state.

| Harness | Arms on | TTL source | Stages | Last verified |
|---|---|---|---|---|
| Claude | mod: `turn.complete` | the transcript in the chat's recorded `native_store`: 3600 or 300, known | push, warn, compact | Replay on installed 0.9.8, 2026-10-14: `ttl=3600`, all three stages for a session that had changed directory. Idle compaction fired live once, 2026-10-12 (session `c64e02c6`, `compact: ok`). |
| Pi | extension: `agent_end` | provider and `PI_CACHE_RETENTION` table in `pi_idle` | push, warn, compact on `anthropic`/`openai` long retention and `openai-codex`; push only otherwise | Replay on installed 0.9.8, 2026-10-14: `openai-codex` 1800 est. with all stages; `xai` push only. |
| Codex | `notify` callback on turn completion, pinned to the recorded thread | `[harness.codex.idle] ttl_seconds` default 1800, estimated | push, warn, compact | Live on installed 0.9.8 with codex-cli 0.162.1, 2026-10-13: two primaries armed with the 1800 s schedule; one real turn on `gpt-5.6-luna` recorded `done.push = sent`. Warn and compact have not fired live on Codex. |
| OpenCode 2.x | V2 sensor: `session.execution.{succeeded,failed,interrupted}` | provider read from `GET /api/session/{id}` at turn end; `openai` → 1800 est., everything else unknown | `openai`: push, warn, compact. Others: push only | **Unreleased (PR #573).** Live on the branch with opencode 2.0.28, 2026-10-14: `openai/gpt-6-luna` → all three stages with 1800 s offsets; `opencode-go/deepseek-v4.1-flash` → push only; steer and queued prompts close the stretch. Meridian-initiated compaction is unit-tested only; the `/compact` route and its event order were seen live through a manual POST. Released ≤0.9.8 arms nothing on 2.x. |
| OpenCode 1.x | frozen V1 sensor: `session.idle` | none (V1 never reports a provider) | push only | Probed live on 1.18.32 when the adapter was built (2026-10-08). Not re-run in phase 2; dispatch to it is covered by unit tests. |
| Cursor | — | — | none | No primary path. |

## Adapter contracts

Runtime probes (P0a–P0d) set these contracts, and G2 hardened them:

- **Claude.** A user return is `prompt.submit` or `command.run` with
  `origin.kind` `composer` or `bridge` (the phone/web client), not
  `turn.start`. `$.session.compact()` is called only from a `$.clock.after`
  callback. The mod can't see its own compaction turn, so it trusts the compact
  promise's result. Between `act` and the compact call it re-checks the return
  epoch and the draft. Under `claude -p` (`isInteractive === false`, no surface)
  it never calls meridian.
- **Pi.** A user return is `input` with `source === "interactive"`. The handler
  cancels timers synchronously and queues `idle return` behind any in-flight
  arm without awaiting the CLI. Pi awaits input handlers, so an awaited CLI
  call cost every prompt about 0.5 s (G2 P1). After `act`, the extension
  re-checks the return revision, `isIdle()`, pending messages and the draft,
  and reports `vetoed` if any changed. Pi's `ctx.compact()` aborts the running
  turn, so compacting over a just-sent prompt would destroy it (G2 P2). Pi
  doesn't expose `compaction.enabled` to extensions, so that guard stays
  unknown for Pi.
- **Codex.** `notify` fires on the app-server/TUI path and is pinned to
  meridian's recorded `harness_session_id`; threads with no stored state are
  dropped. `input-messages` is cumulative, so a return is a growth in its
  length, not a non-empty list. Push and warn are skipped while the pane shows
  Codex working, because Codex has no return signal until the turn ends
  (G2 K2). The current actuator types `/compact`, verifies the text, re-checks
  that the TUI is alive, presses Enter, and reports `ok` only when a newly
  appended `compacted` rollout record is observed. The TUI remains the current
  submission and draft-observation surface; an empty composer appears as soon
  as any turn starts and is no evidence of completion (G2 K1).
- **OpenCode 2.x** (PR #573, v0.9.9; probed on 2.0.28). Events reach the
  sensor through the V2 transport's normalizer, which flattens `data` and keeps
  the envelope `id` as the turn ID. Only events for the launched parent
  `sessionID` count; child task sessions are ignored.
  - *Turn end* is any of `session.execution.succeeded`, `.failed` or
    `.interrupted`; `session.execution.started` marks busy. V2 emits neither
    `session.status` nor `session.idle`.
  - *User return* fires on `session.inbox.enqueued` for an item of type `user`,
    and again on `session.inbox.delivered` for that ID unless Meridian is
    compacting. Enqueue covers a prompt steered into the running execution,
    which never starts a new one. Delivery covers a queued prompt that runs
    after another execution has already armed a stretch. `session.inbox.cancelled`
    drops the ID. Inbox items of type `synthetic`, `compaction` and `move` never
    count as a return.
  - *Compaction* is `POST /api/session/{id}/compact` with `{}`. V2 acknowledges
    admission at once and returns a `compaction` inbox item, so the result
    waits on the stream: `session.compaction.started` for that inbox ID, then
    `session.compaction.ended`, then `session.execution.succeeded`. A
    `.failed` or `.interrupted` terminal after the start resolves as `failed`
    at once. A missing `compaction.started` waits out the 300 s timeout
    (accepted as bounded).
  - Provider and message excerpts are fetched concurrently at turn end from
    `GET /api/session/{id}` (`data.model.providerID`) and
    `/api/session/{id}/message?limit=20&order=desc`. A failed provider read
    leaves the TTL unknown: push only.
  - Context size is not read (`context_tokens` is `None`), so
    `min_compact_tokens` never vetoes. Draft text comes from the attached TUI
    pane.
- **OpenCode 1.x** (frozen). Summarize keeps the session model when the current
  provider and model are supplied. The session GET times out at 10 s and the
  summarize POST at 300 s. The sensor marks its own compaction before the POST;
  the first matching `session.idle` ends compaction classification
  independently of POST response ordering, and failed or cancelled requests
  clear it in `finally`. Messages tied to the compaction part stay classified
  as synthetic.

## Close and reopen

PR #578 (unreleased): before exec, the primary launcher closes the resumed
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
upstream. Most drift fails closed: returns go unseen or compaction is skipped.
These cases fail open or fail silently:

- **Codex busy text `Working (`.** If it changes, `busy` is always false and
  the actuator could type `/compact` into a working session. This fails open.
- **Claude `$.config.list()` key `autoCompact`.** If it is renamed, the mod
  compacts for a user who turned auto-compact off in settings. This fails open;
  core still honours `DISABLE_AUTO_COMPACT`.
- **Return-signal renames** (Claude `origin.kind`, Pi `input.source`, Codex
  cumulative `input-messages`, OpenCode 2.x `session.inbox.*`). Stretches stop
  closing, so notifications stop after the first stretch. This fails silently.
- **Turn-end renames** (Claude `turn.complete`, Pi `agent_end`, OpenCode 2.x
  `session.execution.*`). The sensor never arms: no row, no push, no error.
  The #550 review filed this as "fails closed"; for idle, closed and silent are
  the same thing. OpenCode 2.x did exactly this for every release from 0.9.3 to
  0.9.8 ([lesson](../lessons/harness-integration.md#a-dialect-switch-must-move-every-consumer-of-the-stream)).
- **Reopen close placement** (PR #578). The `close_on_launch` call must stay
  before exec in `run_harness_process`. Moved after the harness starts, Claude
  `recover()` and Pi `restore()` can re-adopt the old stretch first. The call
  also keys the row by `str(harness_adapter.id)` and reads the store under
  `child_env`'s user home; both must match what the harness's own
  `meridian idle` calls use (`claude`, `pi`, `codex`, `opencode`). A mismatch
  closes nothing: a reopened Claude or Pi primary re-adopts the old stretch and
  can compact on its schedule, with no error. This fails silently.
- The Codex tmux actuator narrows the race with a returning user but can't
  close it.
- **TTL keyed by provider, not model** (Pi `openai-codex`, OpenCode `openai`).
  The 1800 s estimate assumes GPT-5.6 or later; an older model on those routes
  inherits it, so warn and compaction can land on a cold cache. This fails
  silently; a configured `ttl_seconds` overrides it
  ([limitation](../decisions/idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd)).

## Related

- [Idle Notifications](../decisions/idle-notifications.md): rationale, rejected alternatives, and the adapter-hosting decision.
- [Idle TTL Detection](../decisions/idle-ttl-detection.md): launch-recorded transcript stores and provider TTL mapping consumed by this mechanism.
- [Harness Adapters](../codebase/harness-adapters.md#idle-bundle-ports): bundle ports and layering boundaries.
- [Harness integration lessons](../lessons/harness-integration.md): upgrade and dialect-drift failure modes.
- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md): provider retention facts behind the TTL defaults.
