# Idle notifications

When a primary session goes idle, Meridian sends a waiting push and, before the
provider's prompt cache expires, a cache warning; on Claude and `agy` the warning can also
prompt the session to record its state. How `agy` gets these, and the state of its
keepwarm, is on its page ([agy idle notices](../harnesses/agy.md#idle-notices)). Harness adapters report what they
observe; core decides what to send and when. The source checkout's
`src/meridian/lib/idle/.context/CONTEXT.md` holds the contract.

## Adapters report facts, core decides

Adapters report facts; core decides scheduling and guards. Separate policy in each
adapter would duplicate precedence and at-most-once handling across harnesses.

Claude and Pi use bundled in-harness extensions; managed Codex and OpenCode primaries
use launcher tasks; `agy`, which Meridian never launches, uses its plugin's MCP server. A running session keeps the extension version it loaded at startup
even after the CLI is upgraded, so an incompatible CLI contract can silently break idle
until restart. Bundling the extension with the CLI keeps their versions coupled at
launch.

## Each stage fires at most once per stretch

Each push and warning fires at most once between user prompts. Finished turns move
pending deadlines but do not re-enable sent stages. Durable state lets adapter reloads
rebuild timers without sending duplicates. The Claude mod's keepwarm pings and state
reminder (below) follow the one warning and leave this rule as it is.

A warning that goes out in the push's place answers the push for the rest of the stretch.
When the warning lands first, the push is dropped; without counting it as answered, the
turn Meridian itself starts after the warning (a keepwarm ping, a state reminder) would re-arm
idle and send the push as a second notice. The user has been told once, and a turn they
didn't start is no reason to tell them again.

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
signed-in Codex and Work retention is not established by that estimate. `agy` has no
detection and gets a default 1800 seconds from observed transcripts. The estimates
are not backend guarantees, and detection by provider can't tell models apart, so an
older model may get an estimate that is too long. A configured `ttl_seconds` wins over
detection and over these defaults. The [retention research](../research/prompt-cache-retention.md) has the
evidence and limits.

Interactive Pi primaries default to `PI_CACHE_RETENTION=long`, unless the user sets it,
so more sessions can get a cache warning. Long Anthropic cache writes cost more;
`short` opts out. Pi's `openai-codex` adapter did not apply the setting in the version
observed.

## Keepwarm pings (Claude)

On an interactive Claude primary, the user can turn on keepwarm: after the cache
warning, the `meridian-idle` mod sends the model a short prompt so the cache stays
warm while the user is away. How it times, submits and stops pings is in the mod's
`README.md` (source checkout `src/meridian/claude_runtime/meridian-idle/`).

- **The warning places ping 1.** `warn_minutes` defaults to 10, so on a one-hour
  Claude cache the warning and the first ping land 50 minutes after the turn. The lead
  is set globally (`[idle] warn_minutes`) or per harness (`[harness.<h>.idle]
  warn_minutes`): one lead can't fit every cache window (a one-hour Claude cache, a five-minute
  Anthropic cache on Pi short retention), and where a TTL is an estimate the user may want more margin.
- **The mod times the later pings, and they send no notice.** Core still fires the
  warning once per stretch on every harness. Why core does not re-claim the warning
  for each ping: `src/meridian/lib/idle/.context/CONTEXT.md`.
- **Never on a cold cache.** A ping on an expired cache pays for the rebuild it was
  meant to avoid. So a ping that cannot go out a minute before expiry is dropped,
  and that ends the stretch's pings.
  A cache shorter than `warn_minutes` gets no warning, since its warning time is
  already past at the turn's end, so a 5-minute cache gets no pings at the default.
- **At most `keepwarm_pings` per stretch (default three, no upper bound; 0 or less turns keepwarm off).** On Anthropic's pricing a cache read costs about
  0.1× the base input price and writing a one-hour cache 2×, so three pings keep the
  cache warm for about 3½ hours for less than one rebuild
  ([Anthropic prompt-caching pricing](https://platform.claude.com/docs/en/build-with-claude/prompt-caching#pricing)).
  The count is the user's spending choice, so it has no upper bound: beyond
  about 19 pings an absence costs more than returning to a cold cache.
- **The user or their config turns it on, never the model.** Keepwarm costs
  money, so it is opt-in: `[idle] keepwarm` (default `false`, overridable under
  `[harness.<harness>.idle]`) sets where every session starts, and
  `/keepwarm on|off` overrides that for the session. The keys are global because
  keepwarm is one preference timed from the provider's cache lifetime; the harness
  is only the adapter that can act on it, Claude today. Core answers the default in `meridian idle config`, so the mod still
  reads no Meridian config itself. A restart, resume or `/clear` returns to the
  config default, because `/clear` starts a new conversation. The model can
  neither turn the mode on nor off.
- **A stretch's pings end on `stop-waiting`, a compaction, or a cold deadline.** Both leave the
  mode on for the next absence. The ping and the denied-tool reason say that
  `stop-waiting` means the model stops waiting for the user and accepts the cache
  going cold, so a model that stops knows what it gives up. A compaction of the
  main conversation ends them because the cache they protected is gone: a ping
  would only prefill the new summary. Subagent compactions and `precompute`
  leave the main cache alone and change nothing.
- **Warming, not working.** The user allowed keeping the cache warm, not resuming
  work, so during a ping's turn the mod denies main-loop tool calls that Claude does
  not mark read-only. Subagent tool calls are not gated. The ping is sent as the plugin, not as the user, so the model does not read
  it as the user being back, and the mod does not count it as a return.
- **The your-turn push keeps the real reply.** A ping's turn re-arms idle like any
  turn, so it can still place a push that was owed. The push quotes the last answer
  from a non-ping turn, not the model's reply to the ping.
- **The transcript is the record.** The mod keeps no log of pings. Each one is in
  the session transcript as a plugin prompt starting
  `[meridian-idle keepwarm <nonce>]`, with the model's reply and any denied tool
  call after it, so whether models actually wait is read from sessions. The ping
  tells the model how to stop and what it costs, but not when stopping is right: a bare
  acknowledgement means it waited, and a denied tool or a reply that starts work
  means it did not.

Keepwarm runs only in the Claude mod; an opt-in `agy` keepwarm is being built
([agy idle notices](../harnesses/agy.md#idle-notices)). A Claude mod can submit a prompt into the
live TUI and deny tool calls during that turn. A Meridian hook outside Claude
could only use `spawn message`, which starts a visible turn through the inbound
gate. Spawned children get no pings ([completion hold](completion-hold.md#waiting-has-one-number)).

## State reminder

When the cache warning fires in an interactive Claude or `agy` primary that has a work item,
the session gets one prompt asking it to bring the work item's recorded state up to
date in place: status, decisions with their reasons, open questions, promises to other
sessions, where it stopped. A resume or a `--from` handoff catches up from that recorded
state first ([continue and fork](../launch/continue-and-fork.md#catch-up-starts-from-recorded-state)),
so the state has to be current when the session goes quiet. Leads resumed days later
found their records behind the conversation: an owner decision nobody was building, an
unanswered question, a merged PR still listed as a draft.

- **At the warning, because the cache is still warm.** The turn reads a cache the
  session was about to lose, so it costs little. With keepwarm on, the reminder takes
  ping 1's place; with keepwarm off it is one unrequested turn.
- **On by default, with its own opt-out** (`[idle] state_reminder`, overridable under
  `[harness.<h>.idle]`): it is one turn per idle stretch on a warm cache, where
  keepwarm's repeated pings are a spending choice the user opts into.
- **Only with a work item.** The work item holds the recorded state; without one there is
  nothing to update.
- **Not a ping.** The reminder turn must be allowed to write files, which ping turns deny,
  and `stop-waiting` means nothing to it.
- **The model may ask to compact (Claude).** A reply whose last line is `compact-now` (the work
  item now holds what resuming needs) makes the mod compact the conversation while the
  cache is still warm, so the user returns to a small context. Pings don't offer it.
- **Core decides, the harness delivers**, as with the rest of idle: the Claude mod
  submits the reminder and compacts; `agy`'s sensor queues it as an inbox message.
  How the reminder rides the single warning without a new `IdleState` field and
  reaches a sensor without core naming a harness:
  `src/meridian/lib/idle/.context/CONTEXT.md`; how the mod times the compaction: the
  mod's `README.md`.

On `agy` the reminder offers no `compact-now`: nothing can compact a running `agy`
conversation without the user typing `/compact`, and the owner's wish for it there
waits on Google ([agy idle notices](../harnesses/agy.md#idle-notices),
[compaction direction](../harnesses/agy.md#compaction-direction)). Pi's idle extension
could carry the reminder but does not.

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

## Compaction only on the model's word, no compaction hooks

Meridian does not compact a session because it went idle; harness-native compaction
settings are separate. A cold return costs a turn, while compaction discards transcript
detail, and session mining can recover older context. The one compaction Meridian runs is
the state reminder's `compact-now`, after the model has recorded its state and judged it
enough to resume from.

Meridian also adds no hook around a harness's own compaction to save state. Claude's
`PreCompact` gives the agent no turn: its output only instructs the summarizer, and
blocking it stalls the session. The owner chose the idle state reminder as the whole of
state-saving.

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
