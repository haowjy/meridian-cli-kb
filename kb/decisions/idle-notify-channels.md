# Decision: Idle Notification Channels

This page owns the notification-side decisions for idle automation: the leaf
package boundary, notification routing through hook events, notice context,
channel secrets, and deliberate spawn notifications. The idle schedule and harness mechanism remain in
[Idle Notifications](idle-notifications.md) and
[Idle Adapter Architecture](../architecture/idle-adapters.md).

## D-idle-leaf-packages — `lib/notify` and `lib/idle` are leaf services

**Decision:** Two new packages, modelled on `lib/artifact/` (a self-contained
feature whose CLI is its only entry point), not on `lib/ops/`.

- **`lib/notify/`** delivers notices. It builds the reason-first session
  context (terminal, actor, and explicit work/project label; see
  [D-idle-notice-context](#d-idle-notice-context--reason-first-titles-and-send-time-cache-countdown))
  and owns the transports: `ntfy`, `smtp`, `gmail` (an SMTP preset), `none`
  (the `command` transport went with
  [D-notify-hook-events](#d-notify-hook-events--notifications-are-hook-events-delivered-by-routes)).
  Since then nothing else calls the transports: a notification is fired as a
  hook event and the built-in `notify` hook calls `send()`; new channels are
  command hooks, not registry entries. `meridian notify` works
  with idle turned off. ntfy titles are RFC
  2047-encoded, because the HTTP `Title` header is Latin-1 and the label's `·`
  would otherwise arrive mangled.
- **`lib/idle/`** holds the policy: a pure timeline, pure guards, a service
  over the state file, and the sidecar loop. It builds `plugin_api.Notification`
  values and hands them to `lib.notify.emit` (until D-notify-hook-events it
  called `lib.notify.send` directly); `lib/notify` never imports `lib/idle`.
  The service's `notify_sender` parameter is only a seam for tests.
- **`lib/state/idle_store.py`** owns the state file, because `lib/state/`
  owns all disk I/O.

`lib/idle` depends on config, state, platform helpers, and the neutral
contracts in `lib/harness/idle_types.py`. **`lib/harness` never imports
`lib/idle`.** An AST test (`test_layering.py`) enforces this, including
function-local imports. Harness modules supply sensors, native idle hooks, env
readers, and TTL detectors through optional `HarnessBundle` ports. The
`NativeIdleHook` groups native observation with downstream delegation:
`meridian idle event` applies a supported event before delegation, but the
delegation runs in `finally` for rejected, malformed, unpinned, blank, or
apply-failing payloads too. The user's Codex `notify` callback is therefore
not consumed by Meridian's parser.

**Why:** `lib/ops/` is policy over spawns, sessions and work and drives launch.
Sending a message touches none of that. Keeping notify separate from idle lets
agents call `meridian notify` without the idle machinery. The harness → idle
ban keeps the dependency one-way. Without it, a harness parser that reads
policy state creates a cycle, and a function-local import hides that cycle
from import-time checks (G2 K3).

## D-idle-notice-context — Reason-first titles and send-time cache countdown

**Decision:** Idle notice
titles lead with the reason, then include the terminal and explicit work
context when available. The shape is `[idle] a2: pi on work: "idle-cache-notify"`;
cache warnings use `[cache warning]`. (Compaction-result titles such as
`[compacted]` went with Meridian's idle compaction,
[D-idle-compaction-removed](idle-notifications.md#d-idle-compaction-removed--meridian-no-longer-compacts-idle-sessions).) The
terminal prefix is the recorded tmux session when present, and is omitted
outside tmux. A work item is labelled explicitly; when there is no work item,
an explicitly labelled project is used when available. `meridian notify`
takes the title from the caller (`meridian notify "<title>" ["<body>"]`) and
always ends the body with the session headline, so the terminal/actor/work
context is never dropped. There is no duplicated tmux footer.

Idle and cache-warning notices calculate the remaining cache lifetime **at send
time**, from the persisted idle anchor and TTL. A known TTL is shown plainly;
an estimated TTL is prefixed `est.`; an unknown TTL has no countdown. The
renderer uses `<1 min until cold` below a minute and `cache cold` after the
deadline. `ttl_estimated` is persisted with the idle state, with
old records defaulting conservatively to estimated. Claude's observed transcript
TTL and Pi's Anthropic retention are known. Pi's `openai` and `openai-codex`
heuristics and OpenCode's `openai` mapping are estimated, and so are explicit,
configured and default TTLs
([D-idle-ttl-detection](idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd)). This changes notice context only;
the schedule and guards are unchanged.

**Why:** A reason-first title tells the user why a notice arrived, while the
terminal and explicit work label identify which session needs attention. Rendering
from persisted state at send time avoids publishing a stale precomputed deadline.
The confidence marker prevents a provider heuristic or configured fallback from
looking like a provider guarantee.

## D-notify-hook-events — Notifications are hook events delivered by routes

**Status:** Decided by the user, 2026-10-17, on
`claude/notify-bash-events-gguu11` (uncommitted when recorded); implements
[#670](https://github.com/haowjy/meridian-cli/issues/670). The user chose
"Full #670" over a lighter "events + named channels" option, and a positional
`meridian notify "<title>" ["<body>"]`.

**Decision:** A notification is a hook event, and the hook rows on that event
deliver it.

- **Three new observe-class events:** `session.responded` (the idle waiting
  push), `session.cache_expiring` (the cache warning) and `notify.manual`
  (`meridian notify`). Idle push/warn and the CLI build one
  `plugin_api.Notification`, the only notification value;
  `lib/notify/emit.py` fires it as its event through `HookDispatcher`, next
  to `spawn.finalized` and `work.done`.
- **Delivery is a built-in hook, `notify`** (`lib/hooks/builtin/notify.py`).
  It calls `lib/notify`'s `send()` directly, lazily, as a companion it wraps,
  the way `git-autosync` uses `autosync_store`. Options (a Pydantic model the
  builtin owns): `channels` (the `push` / `email` aliases or backend names;
  default both) and `priority` (1–5).
- **Default routes are the lowest config layer.** Source `default`, below
  `user`: one row named `notify` per notification event. An unnamed
  `builtin = "notify"` row is named `notify` (the builtin's
  `identity_options` is empty), so a row for an event, even a disabled one,
  replaces that event's default through the ordinary `(name, event)`
  override. Out of the box, delivery is unchanged.
- **Email opt-out belongs to the backend.** A notification with
  `email = false` (`--no-email`, the idle push, `warn_email = false`) skips
  `smtp` and `gmail` however a route names them, not only via the `email`
  alias.
- **Extension channels are ordinary command hooks.** Their JSON stdin carries
  a `notification` object (`title`, `body`, `priority`, `email`, `label`) and
  their env `MERIDIAN_NOTIFICATION_TITLE`, `_BODY` and `_PRIORITY`. A Slack or
  desktop channel is one `[[hooks]]` row plus one script.
- **Idle still records `push|warn: sent|failed`** (the issue's first open
  question). `emit` makes one `SendResult` per route, named by its hook: the
  built-in's failure names each failed transport in its error, and a route
  that delivered nothing is `none`. A stage is marked failed only when a
  route failed and none delivered. If no route ran, the result is ok with a
  warning. The emitter reports route failures, so the dispatcher logs them at
  info rather than printing a second warning.
- **`meridian notify "<title>" ["<body>"]`.** `--title` is removed; the
  session headline always ends the body.
- **Hooks fire without a runtime root.** In an ad-hoc directory nothing has
  initialized, hooks now run; only interval-throttled hooks skip, having
  nowhere to keep their state. `git-autosync` skips there itself.
  `MERIDIAN_HOOKS_ENABLED=false` now also silences notifications.
- **Plugin API 2.0.0.** `lib/hooks` reuses `plugin_api`'s `HookContext`,
  `HookResult` and `HookEventName` (they were identical duplicates).
  `HookContext.runtime_root` is optional, a `notification` field is added, and
  the hook payload is `schema_version` 2. `Notification` validates its own
  title and priority.

**Reshaped after a thermo-nuclear review (2026-10-17, user: "yea fix"):** the
first cut converted `Notice` → `Notification` → `Notice`, returned the
per-transport report as JSON in `HookResult.stdout` (so any command hook
printing `notify_report` was read as delivery results), delivered through a
`plugin_api.notify.deliver()` that imported `lib/notify` and `lib/config`,
appended default routes after layering under a second override rule, kept
the `command` transport (which re-read push/email config and ran both
commands for a push-only route), and gated email only on the `email` alias.
All of that is gone; the bullets above are the result.

**Why:** Two systems overlapped. `lib/notify` had a fixed channel registry
and idle called `notify.send` directly; `lib/hooks` already had events,
source layering, `when` filters, interval throttling and external commands.
"Push me when spawn X finishes", "warnings to Slack, responses to ntfy" or a
new desktop channel each needed core changes. Making notifications events
puts them on the one extension seam Meridian already has.

**Rejected:**

- **`[notify.channels.<name>]` and `[[notify.routes]]`** (the issue's
  proposal). Hook rows already are routes (`event`, `when.agent`, `interval`,
  `enabled`) and command hooks already are channels. A second table would be
  a second config system with its own layering for the same thing.
- **Events plus named channels only** (the lighter option). It would have
  kept idle calling transports directly, so routing and failure reporting
  would still live outside hooks.
- **A fail-open hook with no recorded outcome.** Idle's `done` record would
  lose `push: failed`; folding route results keeps it.
- **A `command` notify backend next to command hooks.** Two ways to run a
  program on a notification; the hook is the one with routing.
- **`plugin_api.notify.deliver()`.** One caller (the built-in), and it made
  the plugin API depend on Meridian's internals.

**Attachments (#666), same branch, asked for by the user alongside this:**
`meridian notify --attach <path>` (repeatable). Paths resolve and must be
readable files when the notice is made, so a bad path fails with exit 2
before any route runs. `Notification.attachments` carries the absolute paths
to every route. Email attaches all of them to the one message, with the type
guessed from the extension. ntfy carries one file per message, so it sends
the text first and then each file as its own message (`PUT` with
`Filename`), rather than moving the body into a header. Meridian sets no size
limit: a server that rejects a file (ntfy.sh's cap, a self-hosted limit)
fails that backend only, naming the file. Text attachments are not also
inlined into the body; one behavior.

**Not done:** structured excerpts as separate fields; a resume /
`tmux attach` hint in the notice; a Python channel entry point in
`plugin_api`. Open: whether channels should see `spawn.finalized` for a
session's own children by default (the issue's second open question).

**Revisit if:** command-hook channels need per-channel state or retries that
hook rows can't express, or a Python channel plugin lands and needs its own
registration.

## D-notify-smtp-password-file — SMTP password from a file, env as leaky fallback

**Decision (U3):** `[notify] smtp_password_file` (a path to a 0600 file)
is the primary mechanism. `MERIDIAN_NOTIFY_SMTP_PASSWORD` is accepted only when
no file is set, and is documented as leaky.

**Why:** `meridian notify` runs inside the harness process (called by the
Claude mod, the Pi extension or an agent), so it sees only the harness child
env. Meridian strips `MERIDIAN_SECRET_*` from every harness child, primaries
included, so the secret mechanism can never reach in-session tooling (see
[Launch System](../architecture/launch-system.md#child-env-boundaries)). A plain
`MERIDIAN_NOTIFY_SMTP_PASSWORD` does get through, but every spawn inherits
it and any agent that runs `env` can copy it into a transcript. A Gmail app
password is a full-mailbox credential.

**Rejected:**

- **`MERIDIAN_SECRET_NOTIFY_SMTP_PASSWORD`:** stripped before the harness
  starts, so in-session email would always fail.
- **The env var as the primary mechanism:** it is inherited by every spawn.
- **ntfy's email forwarding as an email channel:** not built. It would need a
  role-aware result contract.

## D-notify-from-spawns — Spawns may notify; idle automation stays primary-only

**Decision (U2):** Any agent, spawn included, may call `meridian notify`
deliberately. A meridian-base prompt rule says when: blocked on a decision,
long work done, or a failure; not for routine progress. Idle push and warn
run only in primaries.

## Related

- [Idle Notifications](idle-notifications.md): core idle policy, lifecycle, and remaining decision rationale.
- [Idle Adapter Architecture](../architecture/idle-adapters.md): current notify delivery locations and adapter mechanism.
- [Hooks and Plugins](../concepts/hooks-and-plugins.md): hook events, the built-in `notify` route and command-hook channels.
- [Config Precedence](../concepts/config-precedence.md): standard precedence and config-key declarations.
- [Launch System](../architecture/launch-system.md): child-environment secret boundaries.
- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md): cache facts behind notice countdowns.
