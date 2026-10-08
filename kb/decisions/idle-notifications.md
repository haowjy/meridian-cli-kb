# Decisions: Idle Notifications and Idle Compaction

**Status (2026-10-08): settled, not built.** The user accepted the design and
made every open call; no code exists yet. Each section notes where today's
code still differs. Provenance: `work:idle-cache-notify`, `spawn:p7415`
(design), `chat:c7280` (user decisions U1–U6), `chat:c7275` (planning).

## The problem

A primary session sits idle in tmux. When the provider's prompt cache expires,
the next prompt writes the whole context into the cache again. The user wants
three things:

- a phone push shortly after the session goes idle;
- a warning before the cache goes cold;
- if they don't come back, a compaction while the cache is still warm, so the
  eventual return is cheap.

Agents should also be able to ping the user directly. Prior art
(`claude-idle-compact`, `claude-cache`, `claude-cache-keeper`) covers one
harness each and never notifies off the machine.

The default schedule, measured from the last cache write:

| Idle for | Action | Push | Email |
|---|---|---|---|
| 60 s | "waiting on you" | yes | no |
| TTL − 15 min | "cache cold in 15m" | yes | yes |
| TTL − 5 min | compact if every guard passes, then report | yes | no |
| user sends a prompt | cancel everything pending | — | — |

Stages that don't fit inside the TTL are dropped. With a 5-minute cache only the
push fits; with an unknown TTL there is only the push. Which harness has which
TTL is in [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md).

## D-idle-core-decides — Adapters report facts, core decides

**Decision:** Each harness adapter only observes and acts. It reports
facts to core (a turn finished, the user typed a prompt, a timer fired,
whether there is a draft, how many tokens are in context). Core owns the
timeline, every guard, the state file, config resolution and notification
delivery. For compaction, core replies `act` or `skip`. An adapter never reads
meridian config, never writes idle state, never sends a notification itself,
and never compacts without `act`.

Adapters talk to core through a hidden `meridian idle arm|return|fire|done|event`
CLI (in-session TypeScript adapters) or the same API in-process (adapters that
run inside meridian's launcher). Facts visible in env, such as
`DISABLE_AUTO_COMPACT`, `OPENCODE_DISABLE_AUTOCOMPACT` and `PI_CACHE_RETENTION`,
reach core through harness bundle hooks, so the idle package never names a
harness.

**Why:** Four harnesses, each with its own TypeScript or Python adapter. If
policy lived in the adapters, four copies of the guard list and the
at-most-once rule would drift, and a per-tmux-session env override would need
four readers. With one decision point, config precedence lives in one place,
and the hard cases (late timers, reloads, a compaction's own turn, a user
returning mid-compaction) are solved once.

**Rejected:** idle policy inside each adapter.

## D-idle-leaf-packages — `lib/notify` and `lib/idle` are leaf services

**Decision:** Two new packages, modelled on `lib/artifact/` (a self-contained
feature whose CLI is its only entry point), not on `lib/ops/`.

- **`lib/notify/`** delivers notices. It builds the session label
  (`[<tmux session>] <project> · <work id>`) and fans out to a channel
  registry: `ntfy`, `smtp`, `gmail` (an SMTP preset), `command`, `none`. A
  new channel is one module plus one registry entry. `meridian notify` works
  with idle turned off.
- **`lib/idle/`** holds the policy: a pure timeline, pure guards, a service
  over the state file, and the sidecar loop. It depends on `lib/notify`, never
  the reverse.
- **`lib/state/idle_store.py`** owns the state file, because `lib/state/`
  owns all disk I/O.

Both packages depend only on `lib/core`, `lib/config`, `lib/state` and
`lib/platform`. Harness modules never import them; they supply sensors,
event parsers and env facts through types in `lib/harness/idle_types.py`.

**Why:** `lib/ops/` is policy over spawns, sessions and work and drives launch.
Sending a message touches none of that. Keeping notify separate from idle lets
agents call `meridian notify` without the idle machinery.

## D-idle-stretch-at-most-once — Each stage fires at most once per stretch

**Decision:** A *stretch* runs from one user prompt to the next. Push, warn
and compact each happen at most once per stretch. A compaction counts once it
is attempted, whatever the result. Each finished turn inside the stretch
re-anchors the schedule, because the cache was just refreshed. Re-anchoring
moves pending stages later, but never re-enables a stage that already ran.

From the moment core says `act` until 30 s after the adapter reports `done`
(the *compaction window*), plain turn-finished reports are ignored, so the
compaction's own turn can't restart the timeline. A positively identified user
prompt is always honoured, even inside the window. The guard order puts the
cheap checks first and the compaction-only checks last:

1. not a primary
2. idle disabled for this harness
3. stage already done in this stretch
4. stretch closed, or the timer is stale
5. timer fired late (machine slept)

Compaction only, after those:

6. compaction disabled for this harness
7. cache already cold
8. harness busy
9. unsent draft
10. agents running in the harness
11. child spawns still active
12. context under `min_compact_tokens`
13. harness's own auto-compaction off

**Unknown facts (D6, user-accepted):**

| Fact | When unknown | Why |
|---|---|---|
| draft | fail closed: skip compaction | Compacting over someone typing is visible harm. A harness compacts only if it can see the draft. |
| context tokens | fail open: compact | Compacting a small context costs little. |

**State:** one JSON file per harness session at
`~/.meridian/idle/<harness>-<native_session_id>.json`, written atomically and
read truncation-tolerantly, with lazy deletion after 7 days. Timers live in
memory wherever the adapter runs and are re-derived from the file after a reload.

**Why:** A Claude mod hot reload and a Pi `/reload` both wipe in-memory state;
without a durable marker a reload re-sends the push and can compact twice.
The file is the only record (files as authority); timers are disposable
(crash-only).

## D-session-role — One `MERIDIAN_SESSION_ROLE` at the bind seam

**Decision:** Meridian marks every session it launches with
`MERIDIAN_SESSION_ROLE=primary|spawn`. It is set once in
`bind_launch_context()`, next to `_MERIDIAN_HARNESS`, from the existing
primary-launch check, and always overwrites, so a spawn started inside a primary
never inherits `primary`. It is a public handle that propagates to children.
Unset means meridian did not launch the session. It replaces
`_MERIDIAN_PI_SESSION_ROLE`; Pi keeps its internal `primary|spawned` literal
and maps `spawn` to `spawned`. (User decision U5.)

**Why:** Idle automation is for primaries only, and every adapter has to
know the role. All five harnesses pass through the one bind seam, primary and
spawn alike, so a single variable covers them; a per-harness marker would mean
one branch per harness in `launch/`. It is public rather than `_MERIDIAN_*`
because TypeScript adapters, agents and user scripts all read it.

**Caveat:** an agent inside a primary that runs a bare `claude` inherits
`primary`. The Claude mod therefore also requires an interactive TUI surface
before it arms.

**Current code (divergence):** only Pi has a role marker,
`_MERIDIAN_PI_SESSION_ROLE=primary|spawned`, set in
`build_harness_env_overrides` (`lib/launch/env.py`). No TypeScript extension
reads it. See [Pi Lifecycle](../architecture/pi-lifecycle.md#primary-vs-spawned-split).

## D-idle-adapter-hosting — Where each adapter runs

**Decision:**

| Harness | Adapter host | How it gets there | Compacts with | Default |
|---|---|---|---|---|
| Claude | mod inside Claude Code | bundled in meridian-cli, `--plugin-dir` on interactive launches only | `$.session.compact()` | push, warn, compact |
| Pi | a fourth bundled extension | `-e` entrypoint, primary role only | `ctx.compact()` | push only (5 min cache) unless `PI_CACHE_RETENTION=long` |
| OpenCode | task in meridian's attach launcher | optional bundle hook `primary_idle_sensor` | `POST /session/{id}/summarize` with the session's current model | push only (`ttl_seconds` 300) |
| Codex | Codex `notify` command plus a task in the attach launcher | `-c notify=[…]` on the app-server, interactive only | tmux: type `/compact`, check that it landed, then press Enter | push, warn, compact (`ttl_seconds` 1800) |

The launcher task is an asyncio task inside `PrimaryAttachLauncher.run`. It
starts after the TUI is running, is cancelled in the same teardown as the
heartbeat, never writes to stderr (the TUI owns the terminal), and can't
replace the session's real outcome if it crashes.

**Why the launcher, not plugins, for Codex and OpenCode:** both are
managed-attach primaries, so a meridian process already lives for the whole
session, runs an event loop and holds the backend connection. OpenCode needs
no plugin at all: meridian already consumes its raw live events and can POST to
its server. Codex's TUI takes over the observer stream after attach, which
leaves meridian blind to its turns, so Codex additionally needs `notify` as a
sensor. (User decision U5 / design Q4.)

**Why the Claude mod is bundled, not mars-synced:** the mod is code coupled
to the `meridian idle` CLI contract of the same meridian version, not prompt
content. Injecting it with `--plugin-dir` only on interactive launches makes
"primaries only" hold by construction: spawns run `claude -p` and never get
the flag. Claude sessions started outside meridian can opt in through
`CLAUDE_CODE_PLUGIN_DIRS`. (User decision U5.)

**Rejected:**

- **Shipping the mod through `mars sync`:** it would separate the mod's version
  from the CLI it calls.
- **An OpenCode TypeScript plugin:** it would duplicate the event stream
  meridian already reads.
- **A thread from `runner.py` in the PTY path:** a first draft did this; review
  found it needed cross-thread plumbing.
- **A tmux-pane-activity fallback:** not built, because every harness has a
  better sensor.
- **Idle support for the black-box fallback** (managed attach failed): none,
  accepted as a gap.

**Known fragility:**

- The Codex tmux actuator narrows the race with a returning user but can't
  close it.
- Whether Codex `notify` fires for TUI turns when set on the app-server is
  unprobed (P0b). If it doesn't, the next option is a rollout-file watch.
- Claude mod APIs are confirmed only from Claude Code 2.1.295's generated types.

## D-idle-config — Separate `[notify]` / `[idle]` namespaces, standard precedence

**Decision:** New tables `[notify]`, `[idle]` and `[harness.<h>.idle]`. Env
names are spelled out per field: `MERIDIAN_NOTIFY_<KEY>`, `MERIDIAN_IDLE_<KEY>`
and `MERIDIAN_HARNESS_IDLE_<KEY>_<H>`. Keys are named so the key matches the
env suffix (`push_seconds`, `warn_minutes`, `compact_minutes`), not the
planning sketch's longer names (D3).

Precedence compares levels first, then specificity (D4): a per-harness env var
beats the global env var, which beats any file key, per-harness or global, which
beats the default. So `MERIDIAN_IDLE_COMPACT=0|1` exported in one tmux session
always wins over a config file in either direction. There are no CLI flags.

Idle compaction is **not** tied to `primary.autocompact`. That setting is a
context-size threshold with no "off" value. Idle compaction has its own switch
and also respects the harness's own auto-compaction-off setting.

Per-harness idle settings are one generated model per harness
(`harness_idle_model("codex")`), not one shared annotated sub-model. A shared
model would be walked once per harness by the option catalog and raise
`Duplicate config canonical key`, and static metadata can't carry a different
env name per harness. The loader limits behind this are in
[Config Precedence](../concepts/config-precedence.md#declaring-config-keys).

**TTL defaults for harnesses that don't expose one (U4):** Codex
`ttl_seconds = 1800`, OpenCode `300`. Claude and Pi detect the TTL from the
session. The design first proposed 3600 for Codex; research showed GPT-5.6+
caches for 30 minutes, so 3600 was withdrawn.

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

## D-notify-from-spawns — Spawns may notify; idle automation stays primary-only

**Decision (U2):** Any agent, spawn included, may call `meridian notify`
deliberately. A meridian-base prompt rule says when: blocked on a decision,
long work done, or a failure; not for routine progress. Idle push, warn and
compaction run only in primaries.

## D-idle-staging — All four harnesses in one work item

**Decision (U1):** Claude, Pi, Codex and OpenCode ship in one work item, with
adapters in parallel worktrees once shared core merges. The tech-lead's
"Claude and Pi first, Codex/OpenCode as a follow-on" proposal was declined. The
user's standing rule: if non-Claude harnesses are covered at all, cover all of
them.

**Build order:** probes for each harness and four enabling refactors run first:

- **R1:** catalog-driven loading of new config tables, plus the per-harness model
  factory.
- **R2:** the session role variable.
- **R3:** the launcher idle-task hook.
- **R4:** the Claude bundled-mod seam.

After those come notify core, idle core, the `meridian idle` CLI, the four
adapters, the prompt rule, then docs and a cross-harness smoke test.

## Related

- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md):
  the TTL facts behind the defaults.
- [Config Precedence](../concepts/config-precedence.md): standard precedence
  and how config keys are declared.
- [Launch System](../architecture/launch-system.md): the bind seam and the child
  env boundaries.
- GitHub #548: `primary.autocompact` is written into
  `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` as a token count; that variable expects a
  percentage. Found during this design; tracked separately.

## Revisit when

- A probe (P0a–P0d) contradicts an adapter assumption, especially Codex
  `notify` on the app-server.
- A provider changes its default cache retention, or Codex/OpenCode add a
  retention setting.
- A harness gains a native idle or compaction hook that would replace a
  launcher task or the tmux actuator.
