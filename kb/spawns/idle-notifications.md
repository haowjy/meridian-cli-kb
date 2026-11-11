# Idle notifications

When a primary session goes idle, Meridian sends a waiting push and, before the
provider's prompt cache expires, a cache warning. Harness adapters report what they
observe; core decides what to send and when. The source checkout's
`src/meridian/lib/idle/.context/CONTEXT.md` holds the contract.

## Adapters report facts, core decides

Adapters report facts; core decides scheduling and guards. Separate policy in each
adapter would duplicate precedence and at-most-once handling across harnesses.

Claude and Pi use bundled in-harness extensions; managed Codex and OpenCode primaries
use launcher tasks. A running session keeps the extension version it loaded at startup
even after the CLI is upgraded, so an incompatible CLI contract can silently break idle
until restart. Bundling the extension with the CLI keeps their versions coupled at
launch.

## Each stage fires at most once per stretch

Each push and warning fires at most once between user prompts. Finished turns move
pending deadlines but do not re-enable sent stages. Durable state lets adapter reloads
rebuild timers without sending duplicates. The Claude mod's keepwarm pings
(below) follow the one warning and leave this rule as it is.

Reopening a primary through Meridian closes the previous idle stretch before the harness
starts: the user has returned, so the old push and warning are stale. Reloads do not run
the launcher and can rebuild pending timers. Exit hooks alone cannot cover killed
terminals. An in-TUI `/resume`, or a Claude TUI started outside Meridian, has no
launcher and can still adopt the old stretch.

## Running work and the push

Only work that will wake the session holds the push. A certain background wake holds
it; uncertain work delays it. A known wake before cache expiry also makes the cache
warning unnecessary. The response stays owed until the push is sent or the user
returns, so a wake before delivery does not reset the delay.

Treating uncertain work as certain can suppress a needed push forever. A foreground
`meridian spawn wait` can wake the session; detached work, loops, and commands followed
by further work do not establish a wake. See `lib/idle/wake.py`.

Adapters report work only if its completion wakes the session. OpenCode 2.x releases a
hold during a child permission request, because that child needs the user; reconnects
recheck work because the live stream can miss a completion. Codex reported no wake in
two managed-path probes, though one real-use observation disagrees and is unexplained;
it reports no running work until that is reproduced. OpenCode 1.x wake behavior is
unverified.

## Claude's still-waiting signal

Claude's still-waiting permission notification is forwarded as a stateless hook event.
Rebuilding prompt state by correlating tool events failed when hooks rewrote input, a
long approved command stayed active, or an agent asked twice. The notice follows idle
enablement and does not change the stretch.

## Primaries only

Idle automation runs only for primaries. Launch sets `MERIDIAN_SESSION_ROLE` at the
shared bind seam, so a child cannot inherit its parent's role. A bare Claude launched
inside a primary can inherit `primary`; the Claude mod also requires an interactive
TUI. Any agent may still notify deliberately.

## Cache TTL

### Detection reads launch records, never a cwd

The idle service passes the chat's launch-recorded native store to the harness TTL
detector. Claude resolves the live session ID inside that store and reads the latest
assistant cache-creation usage. Looking up the current directory would miss the
transcript after the agent changes directories, because Claude files it under the
launch directory. Missing or invalid identity and unreadable transcripts give an
unknown TTL rather than a guessed location. See
[native session binding](../state/native-session-binding.md).

Pi chooses a TTL from the observed provider and `PI_CACHE_RETENTION`: Anthropic is 300
seconds for short retention and 3600 for long; OpenAI long retention and OpenAI Codex
use an estimated 1800 seconds; unsupported providers and OpenAI short retention are
unknown. OpenCode estimates 1800 seconds for OpenAI and leaves other providers unknown;
signed-in Codex and Work retention is not established by that estimate. The estimates
are not backend guarantees, and detection by provider can't tell models apart, so an
older model may get an estimate that is too long. A configured `ttl_seconds` wins over
detection. The [retention research](../research/prompt-cache-retention.md) has the
evidence and limits.

Interactive Pi primaries default to `PI_CACHE_RETENTION=long`, unless the user sets it,
so more sessions can get a cache warning. Long Anthropic cache writes cost more;
`short` opts out. Pi's `openai-codex` adapter did not apply the setting in the version
observed.

## Keepwarm pings (Claude)

> Not on `main` yet (2026-11-11): built and reviewed, waiting to merge. Delete this
> note once it merges, or the section if it doesn't.

On an interactive Claude primary, the user can turn on keepwarm: after the cache
warning, the `meridian-idle` mod sends the model a short prompt so the cache stays
warm while the user is away. How it times, submits and stops pings is in the mod's
`README.md` (source checkout `src/meridian/claude_runtime/meridian-idle/`).

- **The warning places ping 1.** `idle.warn_minutes` defaults to 10 for every
  harness, so on a one-hour Claude cache the warning and the first ping land 50
  minutes after the turn. The default cannot differ by harness (per-harness idle
  config has only `enabled` and `ttl_seconds`), so Pi, Codex and OpenCode get the
  same 10-minute lead. Where their TTL is an estimate, that is less margin; a user
  who needs more sets `idle.warn_minutes`.
- **The mod times pings 2 and 3, and they send no notice.** Core still fires the
  warning once per stretch on every harness. Why core does not re-claim the warning
  for each ping: `src/meridian/lib/idle/.context/CONTEXT.md`.
- **Never on a cold cache.** A ping on an expired cache pays for the rebuild it was
  meant to avoid. So a ping is dropped when it cannot go out a minute before expiry.
  A cache shorter than `warn_minutes` gets no warning, since its warning time is
  already past at the turn's end, so a 5-minute cache gets no pings at the default.
- **At most three per absence.** On Anthropic's pricing a cache read costs about
  0.1× the base input price and writing a one-hour cache 2×, so three pings keep the
  cache warm for about 3½ hours for less than one rebuild
  ([Anthropic prompt-caching pricing](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#pricing)).
- **Only the user turns it on.** Keepwarm is off by default and the model cannot turn
  it on. It resets on restart and `/clear`, because `/clear` starts a new
  conversation. A reply of `stop-waiting` ends pings for this absence but leaves the
  mode on: the model can neither turn the mode on nor off.
- **Warming, not working.** The user allowed keeping the cache warm, not resuming
  work, so during a ping's turn the mod denies main-loop tool calls that Claude does
  not mark read-only. Subagent tool calls are not gated. The ping is sent as the plugin, not as the user, so the model does not read
  it as the user being back, and the mod does not count it as a return.
- **The your-turn push keeps the real reply.** A ping's turn re-arms idle like any
  turn, so it can still place a push that was owed. The push quotes the last answer
  from a non-ping turn, not the model's reply to the ping.
- **It is an experiment.** Each ping's outcome (stopped, continued or failed, any
  denied tools, a reply excerpt) is kept in the plugin's `$.store`, and
  `/meridian-idle` prints the tally. The ping tells the model how to stop but not
  when stopping is right, so the records show how models read the prompt: a bare
  acknowledgement means the model waited, and a denied tool or a reply that starts
  work means it did not.

Keepwarm runs only in the Claude mod. A Claude mod can submit a prompt into the
live TUI and deny tool calls during that turn. A Meridian hook outside Claude
could only use `spawn message`, which starts a visible turn through the inbound
gate. Spawned children get no pings ([completion hold](completion-hold.md#waiting-has-one-number)).

## OpenCode 2.x sensor

The sensor follows the transport's resolved dialect: the configured preference can be
`auto`, and resolving it separately could disagree with the connection. OpenCode 2.x
has different turn and inbox events from 1.x. A user inbox item counts as a return
both on enqueue and on delivery: a steered prompt may run inside the current execution,
while a queued one can arrive after an idle stretch has started.

A managed primary reconnects its event stream for its whole lifetime; a spawn instead
reconciles a stalled stream and terminates. Treating the primary like a spawn leaves
later turns unobserved.

## Delivery

`lib/notify` and `lib/idle` are leaf services. Delivery doesn't depend on idle policy,
so an agent can notify with idle automation off, and harness adapters supply
observations without depending on idle policy, which avoids a dependency cycle. Native
hook delegation still runs when Meridian can't parse or apply an event, so an existing
user callback is not consumed.

Notifications are hook events delivered by the existing hook routes and command
extension seam, not a second channel with its own routing config. A failed route is
reported and does not stop the lifecycle. Invalid hook rows are dropped one by one:
falling back to defaults could undo a route the user disabled. A partly delivered
notification reports a warning rather than a failure, so a caller doesn't resend text
that already arrived. See
[hooks](../hooks.md) and `docs/hooks.md`.

A notice leads with its reason and session context. A cache countdown is computed when
sent and marks estimates as such, so a provider heuristic doesn't read as a guarantee.

`[notify]` and `[idle]` use standard config precedence; per-harness settings are more
specific within the same level. See `docs/configuration.md` and the source checkout's
`src/meridian/lib/config/AGENTS.md`.

### SMTP password from a file

Why a password file beats the environment: the source checkout's
`src/meridian/lib/notify/AGENTS.md`; why `MERIDIAN_SECRET_*` can't carry it:
`src/meridian/lib/launch/.context/CONTEXT.md` ("Environment and lifetime").

The Gmail transport is tested with a fake; real email delivery is unverified.

## No idle compaction

Meridian does not compact idle sessions; harness-native compaction settings are
separate. A cold return costs a turn, while compaction discards transcript detail, and
session mining can recover older context.

## Known fragility

Adapters depend on upstream event names and payloads. If those change, idle transitions
or background work can go unseen without an error, and a busy signal that stops
matching can cause an early notice. See [upstream drift](../harnesses/overview.md#upstream-behavior-drifts).

- Codex detects busy from the text `Working (`; a change can make notices fire during a turn.
- Claude forwards `classic.Stop` background tasks at `turn.complete`. A missing hook or
  renamed field silently loses the hold; a renamed task type can turn a hold into a
  delay. If `classic.Notification` stops reporting `permission_prompt`, the permission
  notice also disappears silently.
- Return-event drift can leave an idle stretch open and suppress later notices;
  turn-end drift can stop the sensor arming at all.
- Reopen must close the old stretch before the harness starts, using the same harness
  identity and home as its idle commands. Otherwise Claude or Pi can recover the old
  timer and notify on its former schedule without an error.
