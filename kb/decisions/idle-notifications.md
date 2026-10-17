# Decisions: Idle Notifications

## The problem

A primary session sits idle in tmux. When the provider's prompt cache expires,
the next prompt writes the whole context into the cache again. The user wants
two things:

- a phone push shortly after the session goes idle;
- a warning before the cache goes cold.

Until 2026-10-17 a third stage compacted the session while the cache was still
warm, so the eventual return was cheap. It was deleted
([D-idle-compaction-removed](#d-idle-compaction-removed--meridian-no-longer-compacts-idle-sessions)):
old context is recovered by session mining, and each harness compacts on its
own.

Agents should also be able to ping the user directly. Prior art
(`claude-idle-compact`, `claude-cache`, `claude-cache-keeper`) covers one
harness each and never notifies off the machine.

The default schedule, measured from the last cache write:

| Idle for | Action | Push | Email |
|---|---|---|---|
| 60 s after a turn of 120 s or more; 600 s after a quicker or unknown-length turn | "waiting on you" | yes | no |
| TTL − 15 min | "cache cold in 15m" | yes | yes |
| user sends a prompt | cancel everything pending | — | — |

The push delay comes from `[idle] push_seconds`, `long_turn_seconds` and
`quick_turn_push_seconds` (since v0.9.3), so a one-line answer doesn't page the
user a minute later. Work still running when the turn ends can hold or delay
the push and drop the warning
([D-idle-push-in-flight](#d-idle-push-in-flight--the-push-waits-for-work-that-will-wake-the-session-the-warning-does-not)).
Stages that don't fit inside the TTL are dropped. With a
5-minute cache only the push fits; with an unknown TTL there is only the push.
Which harness has which TTL is in
[Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md); what
each harness actually schedules today is in
[Per-harness coverage](../architecture/idle-adapters.md#per-harness-coverage).

## Invariants to keep

- **Fire checks live only in the ordered guards.** `lib/idle/guards.py`
  is one ordered pure function that matches the guard list below line by line.
  A new check goes there, never into an adapter.
- **Running work is read only in core.** Adapters forward the tasks a turn
  left running as `BackgroundTask` facts; `lib/idle/wake.py` alone interprets
  them, including Meridian's own `spawn` CLI grammar
  ([D-idle-push-in-flight](#d-idle-push-in-flight--the-push-waits-for-work-that-will-wake-the-session-the-warning-does-not)).
- **`IdleStore.mutate(fn)` is the only write path.** It locks, reads, applies
  a pure transition and writes atomically, and it is the one seam tests
  replace. `IdleStore.write` exists only as a test convenience; don't add a
  second production write path.
- **The timeline is absolute-time.** `timeline.schedule` is pure and returns
  wall-clock deadlines. That is why an adapter can rebuild its timers from the
  state file after a reload. Relative timers would make reload recovery
  impossible. Recovery serves reloads only: a Meridian reopen closes the
  session's open stretch before the harness starts, so an adapter finds
  nothing to rebuild
  ([D-idle-reopen-return](#d-idle-reopen-return--a-meridian-reopen-is-a-user-return)).
- **Harness isolation.** `launch/` and `lib/idle` contain no `HarnessId`
  branches. Harness modules depend only on `lib/harness/idle_types.py`, never
  on `lib/idle` ([D-idle-leaf-packages](idle-notify-channels.md#d-idle-leaf-packages--libnotify-and-libidle-are-leaf-services)).
  The branch deleted a Pi-specific role branch instead of adding a new one
  ([D-session-role](#d-session-role--one-meridian_session_role-at-the-bind-seam)).
- **Launcher fault containment.** The idle task is cancelled before the
  connection stops and is awaited under `suppress`, so a broken sensor cannot
  change the session's outcome
  ([D-idle-adapter-hosting](#d-idle-adapter-hosting--where-each-adapter-runs)).
- **Idle GC keeps the lock stable.** Stale idle state may be deleted while the
  per-state lock remains in place; GC does not unlink that lock while holding
  it. Lock reclamation, if ever needed, belongs to the validated exclusive
  lock-GC seam, not ordinary idle-state collection.

## D-idle-core-decides — Adapters report facts, core decides

**Decision:** Each harness adapter only observes. It reports facts to core (a
turn finished, the work it left running, the user typed a prompt, a timer
fired, whether the harness is busy). Core owns the timeline, every guard, the
state file, config resolution and notification delivery; a fired timer gets
`act` or `skip` back. An adapter never reads meridian config, never writes
idle state and never sends a notification itself. (While Meridian compacted,
adapters also acted: they compacted only on `act`.)

In-session TypeScript adapters (Claude, Pi) talk to core through a hidden
`meridian idle config|status|arm|return|fire` CLI and pass `--interactive`
on **every** call. Codex's native `notify` callback enters through
`meridian idle event`. An absorbed arm includes its `reason` on the real wire,
so adapters can distinguish preserved timers from a genuine reschedule.
Adapters hosted in meridian's launcher (Codex, OpenCode) call the same service
in-process. Facts visible in env reach core through
harness bundle hooks, so the idle package never names a harness:
`PI_CACHE_RETENTION` reaches it through Pi's `detect_ttl`.
Facts only a sensor can see travel on the event: the OpenCode 2.x sensor sets
`IdleEvent.provider`, and core passes it to the harness's `detect_ttl` without
interpreting the provider name.

**Why:** Four harnesses, each with its own TypeScript or Python adapter. If
policy lived in the adapters, four copies of the guard list and the
at-most-once rule would drift, and a per-tmux-session env override would need
four readers. With one decision point, config precedence lives in one place,
and the hard cases (late timers, reloads, a compaction's own turn, a user
returning mid-compaction) are solved once.

**Rejected:** idle policy inside each adapter.

## D-idle-stretch-at-most-once — Each stage fires at most once per stretch

**Decision:** A *stretch* runs from one user prompt to the next. Push and
warn each happen at most once per stretch. Each finished turn inside the
stretch re-anchors the schedule, because the cache was just refreshed.
Re-anchoring moves pending stages later, but never re-enables a stage that
already ran. An arm on a closed stretch always opens the next one. A user
return is always honoured: a positively identified prompt, or an arm that
carries `--implies-return` (Codex's only return signal).

The guard order puts the cheap checks first:

1. not a primary
2. idle disabled for this harness
3. stage already done in this stretch
4. stretch closed
5. timer is stale (the anchor moved)
6. stage not scheduled
7. timer fired early
8. timer fired late (machine slept)

Seven compaction-only guards followed these (compaction disabled, cache
already cold, harness busy, agents running, child spawns active, not worth it,
harness auto-compaction off); they were deleted with
[D-idle-compaction-removed](#d-idle-compaction-removed--meridian-no-longer-compacts-idle-sessions).
Running work now shapes the schedule at arm time instead of vetoing at fire
time. The Codex sidecar still skips a fire while the pane shows Codex working
([adapter contracts](../architecture/idle-adapters.md#adapter-contracts)).

**State:** one JSON file per harness session at
`~/.meridian/idle/<harness>-<native_session_id>.json`, written atomically and
read truncation-tolerantly, with lazy deletion after 7 days. Timers live in
memory wherever the adapter runs and are re-derived from the file after a reload.

**Why:** A Claude mod hot reload and a Pi `/reload` both wipe in-memory state;
without a durable marker a reload re-sends the push (and, while Meridian
compacted, could compact twice).
The file is the only record (files as authority); timers are disposable
(crash-only).

## D-idle-reopen-return — A Meridian reopen is a user return

**Status:** merged in PR #578 (`b90b78517`), released in v0.10.3.

**User rule (2026-10-14):** "if we close it, we should get rid of any
autocompaction" / "closing it and reopening it - we shouldn't autocompact it".

**Problem:** closing a primary only cancelled in-memory timers (Claude
`session.end`, Pi `session_shutdown`, launcher sidecar teardown), and nothing
closed the stretch. On reopen, Claude's `recover()` and Pi's `restore()`
rebuilt the stored open stretch, the same path that serves reload. The old
compaction then ran if the cache was still warm and the fire was not late.
Codex and OpenCode never re-adopt a stored stretch, so they were not affected.

**Decision (D6):** Meridian owns resume for every harness: each adapter
refuses the native resume flags, and the launcher knows the harness session
id before exec. So a reopen is treated as what it is, a user return. In
`run_harness_process`, after `prepare_prelaunch` and before the harness
starts, the launcher calls `close_on_launch(harness, session, env)`
(`lib/idle/service.py`). If that session has an open stretch, it applies the
same close transform as `meridian idle return` (`_close_open_stretch`:
`stretch_open = false`; at the time it also cleared the compaction window and
expected-compaction flag, which no longer exist). Claude and Pi then find nothing open to
re-adopt, and the first finished turn opens a fresh stretch.

- No new state, guard, or arm/fire branch. The store schema is unchanged.
- It never creates a row. A fresh launch also has a session id before exec
  (Claude mints one), and finds no row, so the call does nothing. An
  already-closed row is left untouched.
- It never fails the launch: errors are logged at debug and swallowed.
- It does not depend on how spawn ids are assigned on resume.

**Scope (D2):** the whole stretch ends, not only compaction. A push or warning
for a stretch the user has already come back to is meaningless too.

**Reload recovery holds** because a reload (Claude mod hot reload, Pi
`/reload`) runs no launcher. Rebuilding timers from absolute deadlines stays
the reload path.

**Accepted gaps:**

- Switching sessions inside a running Claude TUI with `/resume` runs no
  launcher. `recover()` can adopt that session's still-open stretch until its
  deadlines lapse or the next prompt closes it. If this ever matters, the fix
  belongs in the Claude mod (compare against the launch's recorded session),
  not in core.
- A Claude TUI started outside Meridian (opted in through
  `CLAUDE_CODE_PLUGIN_DIRS`) has no launcher, so a reopen still rebuilds the
  old stretch.
- A killed primary that is never reopened keeps its row in `idle status` until
  a reopen or the 7-day lazy deletion. No timer exists to fire it.

**Rejected:**

- **Launch-owned stretches (D1, an earlier revision of PR #578).** `arm`
  recorded `owner = MERIDIAN_SPAWN_ID`; a fire from a different owner skipped
  `owner-gone` and closed the stretch, and an arm from a different owner
  opened a new one. It worked only because every primary launch, resume
  included, mints a new spawn id. The user does not want that as a contract
  ("I don't think every launch should get a new spawn id"); if a resumed
  primary keeps its spawn id, the reopen matches the old owner and the bug
  returns silently. [D-spawn-run-identity](spawn-identity.md#d-spawn-run-identity)
  later decided that it does. #616, which built that reopen, closed unmerged; identity-v2 keeps the requirement. It also added about five concepts (owner identity, a
  spawn-id dependency, an `owner-gone` reason that mutates state, cross-owner
  arm branches, a lazy close) to every harness for a bug that lives in two
  adapters.
- **Exit-side close** in launcher teardown, Claude `session.end` or Pi
  `session_shutdown`: a killed terminal, `tmux kill-session` or SIGHUP runs no
  exit hook, and `session.end` may also fire on reload.
- **Per-adapter "fresh start closes" logic** in `recover()`/`restore()`: the
  engines cannot tell a reload from a resume, and reload recovery must keep
  working.

**Verified:** live probe on `ae3636a22` (Claude `claude-haiku-4-5`, Pi
`deepseek/deepseek-flash`, TTL forced to 600 s). Claude left open: compaction
ran. Claude killed and reopened with nothing typed: the row closed about 1.4 s
after the reopen started, `done` stayed `{}` past the old compact deadline, no
compaction marker. A new turn opened stretch 2 on a fresh schedule. Pi killed
and reopened: closed about 1.7 s after reopen, no compaction. A fresh Claude
launch created no row. Codex and OpenCode pass the same launcher call but had
nothing to reproduce. A later probe (`spawn:p7893`) confirmed Codex and
OpenCode: the old row closed at reopen and nothing fired. Per-adapter
close/reopen paths are in
[Idle Adapter Architecture](../architecture/idle-adapters.md#close-and-reopen).

## D-session-role — One `MERIDIAN_SESSION_ROLE` at the bind seam

**Decision:** Meridian marks every session it launches with
`MERIDIAN_SESSION_ROLE=primary|spawn`. It is set once in
`bind_launch_context()`, next to `_MERIDIAN_HARNESS`, from the existing
primary-launch check, and always overwrites, so a spawn started inside a primary
never inherits `primary`. It is a public handle that propagates to children.
It is the sole role marker; Pi's prelaunch maps `spawn` to its internal
`spawned` launch profile (see
[Pi Lifecycle](../architecture/pi-lifecycle.md#primary-vs-spawned-split)).
(User decision U5.)

Core resolves the role per call:

| `MERIDIAN_SESSION_ROLE` | `--interactive` on this call | Idle role |
|---|---|---|
| `primary` | either | primary |
| `spawn` | either | spawn: always disabled |
| unset | yes | primary: a Claude TUI started outside meridian that opted in |
| unset | no | none: disabled |

**Why:** Idle automation is for primaries only, and every adapter has to
know the role. All five harnesses pass through the one bind seam, primary and
spawn alike, so a single variable covers them; a per-harness marker would mean
one branch per harness in `launch/`. It is public rather than `_MERIDIAN_*`
because TypeScript adapters, agents and user scripts all read it. The flag is
per call, not only on `config`: core checks the role on every `arm` and `fire`
(and `done`, while it existed). When only `config` carried it, an outside-meridian Claude session read
`enabled: true` and then had every arm declined (G2 C1).

**Caveat:** an agent inside a primary that runs a bare `claude` inherits
`primary`. The Claude mod therefore also requires an interactive TUI surface
before it arms.

## D-idle-adapter-hosting — Where each adapter runs

**Decision:** The current adapter hosts, bundle ports, per-harness schedules,
contracts, close/reopen behavior, and known fragility are maintained in
[Idle Adapter Architecture](../architecture/idle-adapters.md). Claude and Pi
use bundled in-harness extensions; Codex and OpenCode use tasks in Meridian's
managed attach launcher. The hosting boundary keeps harness observation and
actuation at the adapter while core owns idle policy.

**Why:** The launcher is the right host for Codex and OpenCode because both are
managed-attach primaries: a meridian process already lives for the whole
session, runs an event loop and holds the backend connection. OpenCode needs no
plugin at all: meridian already consumes its raw live events and can POST to
its server. In the current Codex attach path, the TUI/observer arrangement
leaves Meridian dependent on `notify` for the turn sensor; the tmux pane now
serves only the busy probe. (User decision U5 / design Q4.)

**Bounded Codex backend probe (history; the tmux compaction actuator it was
weighed against was deleted with
[D-idle-compaction-removed](#d-idle-compaction-removed--meridian-no-longer-compacts-idle-sessions)):** On `codex-cli
0.162.1`, a second initialized WebSocket client resumed a thread while the
native TUI was attached and successfully called `thread/compact/start`. Both
WebSocket clients and the TUI remained usable, and the TUI rendered the
completion. This corrects any categorical claim that backend compaction is
impossible, but production kept compacting through tmux: approval routing was
not exercised, and the actuator was never switched. A backend actuator would
remove the screen-gated composer clear
([D-idle-compact-over-drafts](idle-compaction-actuator.md#d-idle-compact-over-drafts--compaction-wins-over-unsent-text)),
but nobody has evaluated it as a replacement.
Provenance: `evidence/codex-backend-compact-probe.md` (`work:idle-cache-notify`).

**Why the Claude mod is bundled, not mars-synced:** the mod is code coupled
to the `meridian idle` CLI contract of the same meridian version, not prompt
content. Injecting it with `--plugin-dir` only on interactive launches makes
"primaries only" hold by construction: spawns run `claude -p` and never get
the flag. Claude sessions started outside meridian can opt in through
`CLAUDE_CODE_PLUGIN_DIRS`. (User decision U5.)

**Upgrading under running sessions:** the version coupling applies per session.
A session loads the mod (or Pi extension) at start and keeps it after
meridian-cli is upgraded. If the upgrade changes the `meridian idle` flags or
reply shape, that session's idle silently stops until it restarts. v0.9.8 did
this when it removed `--cwd`; see
[Bundled mods outlive the CLI they call](../lessons/harness-integration.md#bundled-mods-outlive-the-cli-they-call).

**Rejected:**

- **Shipping the mod through `mars sync`:** it would separate the mod's version from the CLI it calls.
- **An OpenCode TypeScript plugin on `session.idle`:** it would duplicate the event stream meridian already reads, and the launcher can POST the compaction directly.
- **A thread from `runner.py` in the PTY path:** a first draft did this; review found it needed cross-thread plumbing.
- **Codex hooks as the return sensor:** they need an interactive trust screen.
- **A spawn → session index for Codex:** meridian's recorded `harness_session_id` already is the main thread id.
- **A tmux-pane-activity fallback (D5):** not built, because every harness has a better sensor.
- **Idle support for the black-box fallback** (managed attach failed): none, accepted as a gap.
- **Sharing adapter code between the Claude mod and the Pi extension:** the host APIs differ (`$.process.run` against `node:child_process`). The duplicated reply handling is better removed by having core return the pending stages from `arm` and `status` (GitHub #549; see [Structural debt](#structural-debt)).

## D-idle-config — Separate `[notify]` / `[idle]` namespaces, standard precedence

**Decision:** New tables `[notify]`, `[idle]` and `[harness.<h>.idle]`. Env
names are spelled out per field: `MERIDIAN_NOTIFY_<KEY>`, `MERIDIAN_IDLE_<KEY>`
and `MERIDIAN_HARNESS_IDLE_<KEY>_<H>`. Keys are named so the key matches the
env suffix (`push_seconds`, `warn_minutes`), not the
planning sketch's longer names (D3).

Precedence compares levels first, then specificity (D4): a per-harness env var
beats the global env var, which beats any file key, per-harness or global, which
beats the default. So `MERIDIAN_IDLE_ENABLED=0|1` exported in one tmux session
always wins over a config file in either direction. There are no CLI flags.
`meridian idle config --harness <h>` takes precedence over `_MERIDIAN_HARNESS`
when choosing which harness's settings to resolve.

Idle has no relation to `primary.autocompact`, a context-size threshold for
the harness's own compaction. Meridian's idle compaction, which had its own
`compact*` keys, was deleted
([D-idle-compaction-removed](#d-idle-compaction-removed--meridian-no-longer-compacts-idle-sessions)).

Per-harness idle settings are one generated model per harness
(`harness_idle_model("codex")`), not one shared annotated sub-model. A shared
model would be walked once per harness by the option catalog and raise
`Duplicate config canonical key`, and static metadata can't carry a different
env name per harness. The loader limits behind this are in
[Config Precedence](../concepts/config-precedence.md#declaring-config-keys).

**TTL defaults for harnesses that don't expose one (U4):** Codex
`ttl_seconds = 1800`. Claude reads the transcript TTL; Pi and OpenCode 2.x map
the session's provider to a TTL. The detectors are covered in
[D-idle-ttl-detection](idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd).
*Superseded (PR #573, v0.9.9):* OpenCode had a `ttl_seconds = 300`
default. A configured TTL beats detection, so the default hid the provider and
scheduled nothing past the push
([OpenCode provider TTL](idle-ttl-detection.md#opencode-provider-ttl)).

## D-pi-cache-retention — Meridian launches Pi primaries with long cache retention

**Decision (U9, `chat:c7347`):** Meridian sets `PI_CACHE_RETENTION=long` in
the child env of every interactive Pi primary it launches. A user value from
the shell, profile env or project env always wins. A blank value counts as
missing and is filled. Spawns and non-interactive launches never get the
default. It is launch behaviour, independent of `[idle]`: a Pi primary gets
the one-hour Anthropic cache (or OpenAI's extended retention) even with idle
notifications off. `PI_CACHE_RETENTION=short` is the opt-out.

**Why:** The idle rule is the same on every harness; the cache TTLs are not.
Pi's 5-minute default left room for the push only, so warning and compaction
never fit (see the schedule above). Pi's retention switch is the only cache
lever Meridian can pull from outside a harness. The user asked why the four
harnesses behaved differently out of the box and chose consistency.

**Cost:** Anthropic prices one-hour cache writes above five-minute writes
([Anthropic retention](../research/prompt-cache-retention.md#anthropic-retention-which-opencode-and-pi-inherit)).
The tech-lead had left the default to the user for this reason. It ships as a
spend change: the CHANGELOG upgrade notes and `docs/upgrading.md` name it and
give the opt-out.

**Seam:** the harness adapter's `env_defaults(config, *, run)` is launch-aware.
`lib/launch/env.py::build_harness_child_env` calls it after inherited and
runtime values are bound and writes a key only when the child value is missing
or blank. Pi adds `PI_CACHE_RETENTION=long` when `run.interactive`, next to its
agent-dir and session-dir defaults. Forced adapter policy stays in
`env_overrides()`. See [Launch System](../architecture/launch-system.md#child-env-boundaries).

**Not covered, by evidence:**

- **Pi `openai-codex`** is not changed by the retention default. Pi 1.1.0's
  `openai-codex-responses.js` sends only `prompt_cache_key` and never applies
  `PI_CACHE_RETENTION`. Idle on that provider still schedules the warning,
  from an estimated 1800 s
  ([D-idle-ttl-detection](idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd)).
  *Superseded (2026-10-13):* this page used to keep `openai-codex` push-only,
  reasoning that warning and compaction "would target a cache that isn't
  there". That conflated "no retention knob" with "no cache". The ChatGPT
  backend caches automatically by `prompt_cache_key`, the same way it does for
  the Codex harness. Normalising `openai-*` IDs to `openai` stays withdrawn,
  because Pi does not extend retention there.
- **OpenCode** has no retention switch Meridian can set; its 5-minute
  Anthropic cache is hard-coded. Its `anthropic` sessions stay push-only. Its
  `openai` sessions schedule the warning from an estimated 1800 s
  ([OpenCode provider TTL](idle-ttl-detection.md#opencode-provider-ttl)).

**Rejected:**

- **Leaving Pi on short retention** and documenting `PI_CACHE_RETENTION=long`
  as an opt-in: cheaper per write, but Pi primaries would stay the one harness
  where only the push fires.
- **A separate `apply_env_defaults(child_env, *, run, config)` hook**
  (`2c7c7f78`): review rejected it as a second seam for one concept. Folding
  `run` into `env_defaults` keeps one place where adapter defaults live and one
  missing-or-blank rule in `build_harness_child_env` (`bac44c01`).

## D-idle-push-in-flight — The push waits for work that will wake the session; the warning does not

**Decision (Claude primaries first):** The waiting push and the cache warning are
two independent notices with one shared stretch. When a main-loop turn ends,
the adapter forwards the work still running as facts in the host's own terms,
mapped to a kind (`agent`, `shell` with its command line, `watcher`, `other`).
Core alone reads them (`lib/idle/wake.py`), reduces them to one expected wake,
and `schedule()` places both stages from it:

- **Certain wake → no push this anchor.** Subagents and workflows (`agent`) and
  shells running `meridian spawn …` end and wake the session; that turn arms
  again. The response stays owed: the stretch keeps `push_owed_since_ms` (set
  when the user last spoke), so the push on the wake turn gets the 60-second
  long-turn delay rather than the 10-minute quick-turn one.
- **Known wake time → the warning only if the cache goes cold first.** A
  `meridian spawn wait` yields by its `--yield-after-secs`, else the harness's
  wait-yield default (3000 s for Claude). If that lands before the cache
  expires, the wake keeps the cache warm and the warning is dropped; a longer
  yield keeps it.
- **Maybe → the push is delayed, not held.** Any other background task may wake
  the session or run for good (a dev server), so the push waits at least
  `quick_turn_push_seconds` (600 s).
- **Watchers are ignored**: they may stand forever and wake only on events.

Otherwise the warning arms on every turn, held push or not.

**Why:** Before this, the Claude mod skipped the whole arm while a subagent ran
(losing the cache warning with the push), and background shells were
invisible, so a primary that ran `meridian spawn --bg` plus a backgrounded
`spawn wait` paged the user while the session was about to resume itself. The
push means "the agent is done and waiting on you"; work that will wake the
session makes that false. The cache cost of a cold return is real no matter
what is running, so the warning stays unless a known wake beats expiry.

**Why the adapter forwards the raw list (thermo review, 2026-10-17):** a first
cut had the mod sort tasks into four counts with its own `meridian spawn` regex.
Review found that copy already disagreed with Pi's spawn-watch regex
(`meridian -C dir spawn wait`), that a count cannot carry a per-call
`--yield-after-secs` (a 7200 s wait wrongly dropped the warning), and that
`--agents-running` doubled as an "adapter senses work" flag. Meridian's own CLI
grammar is parsed once, in core. The same review deleted a spawn-store check
that held the push while child spawns were active: a detached `spawn --bg` with
nothing waiting never wakes the session, so it held the push until the user's
next prompt, and a stale active record held every push. It also replaced a
persisted `push_held` reason plus a previous-anchor override with the single
`push_owed_since_ms` timestamp.

**How Claude senses it:** the only host surface that lists background work is
the `background_tasks` of the classic `Stop` hook, which a mod hooks as
`classic.Stop`. Probed on Claude Code 2.1.302: `Stop` fires just before the
main loop's `turn.complete`, lists running shells (with their command line),
subagents and workflows, and the task's completion arrives as a
`task-notification` prompt that runs another main-loop turn with an empty list.
The mod forwards that list with `idle arm --background-tasks`; a turn with no
`Stop` forwards nothing and gets the ordinary push. Other harnesses report no
tasks yet and keep the ordinary push (#669). `session_crons` are not forwarded.

**Gap:** a `meridian spawn --bg` with nothing waiting on it is invisible (its
shell exits at once), so the push fires normally. Agents pair `--bg` with
`spawn wait`, which is visible.

## D-idle-compaction-removed — Meridian no longer compacts idle sessions

**Decision:** Idle compaction is deleted end to end: the compact stage and its
timeline slot, the guards and break-even (`worth.py`), the claim/done states,
context baselines, `idle done`, the compaction notices, the Claude mod's
`$.session.compact()` path, the Pi extension's `ctx.compact()` path, the Codex
tmux actuator, both OpenCode compaction calls, the `compact_costs` and
`autocompact_off` bundle ports, and every `compact*` config key and env var.
Idle is the push and the warning. Supersedes D-idle-compact-default-off (made
the same day: opt-in behind `idle.compact = false`) and, for Meridian's own
compaction, D-idle-compaction-growth-floor and D-idle-compact-over-drafts.

**Why:** The warning already tells the user the cache is going cold; a cold
return costs one expensive turn, while a Meridian compaction loses transcript
detail for good. Old context is cheaper to recover with session mining
(`--from`) than to keep warm by compacting. Claude Code 2.1.299+ ships its own
idle compaction (200K+ tokens, about 54 minutes idle, `/config` → Idle
compaction), and every harness compacts on its own; Meridian leaves those
settings alone. Keeping roughly 1.3k production lines behind a default-off
flag meant every arm still branched on compaction states, and a disabled
compaction still placed a timer and spawned a CLI call per stretch only to be
rejected by its guard (thermo review, 2026-10-17).

**Kept:** OpenCode 1.x still classifies its own compaction messages so a native
`/summarize` or autocompaction is not mistaken for a user return. Sensors
expose `busy()`; the sidecar consults it for external-event sensors (Codex) to
skip a push or warning while the TUI is mid-turn.

## Structural debt

| Debt | Tracked in |
|---|---|
| Core never returns the stages still pending, so the idle protocol is written four times (sidecar, Claude mod, Pi extension, CLI). Fix: `arm` *and* `status` return them. | GitHub #549 (widened by F1) |
| Policy assembly is resolved in `IdleService`; remaining work is grouping the loose idle capabilities and giving sidecar reporter/task lifecycle one owner. | GitHub #554 (residual grouping and sidecar ownership) |
| State machine lift to `machine.py`; launcher-owned sensor lifecycle; split of `IdleSensor`; dedicated Codex identity pin; arm-drift diagnostic | `src/meridian/lib/idle/.context/FUTURE` |
| `effective()` re-derives the option catalog by hand; idle/notify config models belong in their own modules | `src/meridian/lib/config/.context/FUTURE` |
| `== "idle"` literals switch the JSON error envelope in `main.py` | `src/meridian/cli/.context/FUTURE` |
| Claude-only `claude_runtime` touches six projections; two tmux capture helpers; V2 sensor module split; `events()` drain split | `src/meridian/lib/harness/.context/FUTURE` |
| Channels don't know whether they serve push or email | `src/meridian/lib/notify/.context/FUTURE` |

## Deferred adjacent issues

These requests were found during idle-notification testing but are not delivered
by this decision or its implementation:

- **GitHub #557:** crash recovery can log an expected native-snapshot deferral
  with a traceback into a resumed primary TUI. It remains a separate recovery
  and ownership issue; the original crash cause is unknown.
- **GitHub #558:** session browse may gain tmux labels and a recently-crashed
  promotion group. The requested promotion expiry is **24 hours from the first
  time the crash is shown in `meridian session browse`**, not from crash
  detection or background indexing; later views must not reset it. This remains
  a deferred issue, not a delivered browser feature.

## Verification gaps

**The Gmail channel has never sent a real email.** Every live smoke used ntfy.
The gmail path (`smtp.gmail.com:587`, STARTTLS, app password read from a 0600
`smtp_password_file`) is covered only by unit tests with a fake transport. A
first real send could still fail on login or TLS.

To close the gap, the user sets up an app password and sends one test notice:

1. Turn on 2-Step Verification for the Google account. App passwords are
   unavailable without it.
2. Create an app password for Mail.
3. Save it to `~/.meridian/smtp.pass` and `chmod 600` the file.
4. Set `email_to`, `smtp_user` and `smtp_password_file` under `[notify]`
   (`email_backend` already defaults to `gmail`).
5. Run `meridian notify "test" --json`. The report's `gmail` result should
   carry no `error`, and the message should arrive in the inbox.

**The Claude running-work hold has no recorded live probe.** The
`classic.Stop` → `idle arm --background-tasks` path is covered by the mod's
`register.test.ts` and `tests/unit/idle/test_wake.py`.

## Related

- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md):
  the TTL facts behind the defaults.
- [Idle TTL Detection](idle-ttl-detection.md): launch-recorded transcript stores
  and provider TTL mapping.
- [Config Precedence](../concepts/config-precedence.md): standard precedence
  and how config keys are declared.
- [Launch System](../architecture/launch-system.md): the bind seam, the child
  env boundaries and the primary idle sidecar.
- [Harness Adapters](../codebase/harness-adapters.md#idle-bundle-ports): the
  idle bundle ports.
- [Idle Adapter Architecture](../architecture/idle-adapters.md): current
  adapter locations, coverage, contracts and fragility.
- GitHub #548: `primary.autocompact` is written into
  `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` as a token count; that variable expects a
  percentage. Found during this design; tracked separately.

## Revisit when

- A harness update invalidates one of the runtime-probed event contracts in [Idle Adapter Architecture](../architecture/idle-adapters.md#adapter-contracts), especially a fail-open one. Re-run the turn-end probe on
  every harness major version: a renamed turn-end event fails silently.
- Meridian drops OpenCode 1.x: delete the V1 sensor and its dispatch branch.
- A provider changes its default cache retention or the price of long cache
  writes, or Codex/OpenCode add a retention setting.
- Pi applies `PI_CACHE_RETENTION` in its `openai-codex` adapter, or renames or
  drops the variable.
- OpenAI publishes a ChatGPT-backend cache TTL, or observed cache hits after
  idle contradict the 1800 s estimate for Codex, Pi `openai-codex` and OpenCode
  `openai`. The baseline is the 2026-10-14 probe
  ([measured idle window](../research/prompt-cache-retention.md#measured-idle-window-2026-10-14)):
  Pi hit at 25 and 35 min and missed at 15, 45 and 60; OpenCode was
  confounded. The backend's window varies with routing and load, so a
  measurement is an observation under that hour's conditions, not a value
  to confirm; re-measure only to check that 1800 s is still inside the
  window most of the time, with a distinct prefix per gap so no resend
  refreshes another gap's entry.
- Claude files a `/clear` successor outside the launch store, or Meridian stops
  recording `native_store` on the chat; Claude TTL detection would then return
  unknown.
- A harness gains a native idle hook that would replace a launcher task, or
  Claude Code exposes background work somewhere other than the classic `Stop`
  hook's `background_tasks`.
- The `arm` contract changes next (this also breaks idle in sessions already
  running when the upgrade lands), or a fifth adapter is about to be written.
  Do #549 first (pending stages from `arm` and `status`), plus the Codex
  identity-pin op from `lib/idle/.context/FUTURE`, so the new adapter
  doesn't become a fifth copy of the protocol.
- A new idle capability needs a bundle port. Group the ports first (#554).
- External Claude TUIs or in-TUI `/resume` become workflows that matter: no
  launcher runs there, so a reopen can still adopt the old stretch
  ([D-idle-reopen-return](#d-idle-reopen-return--a-meridian-reopen-is-a-user-return)
  gaps). The fix belongs in the Claude mod.
- Codex changes its `Working (` busy text: the sidecar's busy probe would
  read false and push or warn while Codex works
  ([known fragility](../architecture/idle-adapters.md#known-fragility)).

## Sources

- `work:idle-cache-notify`: `design/`, `DIVERGENCE/`, `reviews/`, and `evidence/`
  hold implementation history, rejected mechanisms, and verification records.
- `chat:c7280`: user decisions U1–U6; `chat:c7347`: Pi retention decision U9.
- `work:idle-ttl-detection` (`chat:c7500`, branch `fix/idle-ttl-detection`):
  before-state evidence and D1–D5 in `decisions.md`, Pi provider research
  `research/pi-provider-cache-ttl.md` (`spawn:p7724`), review
  `reviews/review-1.md`, probe `evidence/probe-ttl-detection.md`.
- `work:idle-ttl-detection` phase 2 (branch `fix/opencode-idle-v2`, PR #573):
  `decisions.md` "2026-10-14 phase 2" (D6–D8); probes
  `evidence/probe-opencode-v2-events.md` (`spawn:p7758`),
  `evidence/probe-codex-idle.md` (`spawn:p7759`),
  `evidence/probe-codex-idle-turn.md` (`spawn:p7767`),
  `evidence/probe-opencode-idle-gate.md` (`spawn:p7788`); reviews
  `reviews/review-2.md` … `review-4.md`; live fix checks
  `evidence/coder-fix-round-2-live.md`, `evidence/coder-fix-round-3-live.md`.
- `work:idle-close-no-compact` (PR #578, branch `fix/idle-close-no-compact`):
  `decisions.md` D1–D6 (D6 is the shipped design), `explore-close-reopen.md`
  (`spawn:p7881`), `reviews/review-2-thermo.md` (`spawn:p7889`, why D1 was
  replaced), `evidence/probe-reopen-return.md` (`spawn:p7891`),
  `pr-body-idle-close.md`. D1's review and probe (`reviews/review-1.md`,
  `evidence/probe-close-reopen.md`) describe the replaced design. The same work
  item's D7/D8 and `investigate-opencode-compact-completion.md` cover the
  OpenCode 2.x stream lifetime (PR #583).
- `work:idle-compaction-floor`: `decisions.md` "2026-10-15 User: compact even
  with a draft" and `evidence/drop-draft-guard.md` (Codex source reading at
  `rust-v0.162.1`, live probes `spawn:p7991`, `spawn:p7992`, `spawn:p8002`,
  `spawn:p8003`).
- Process lessons: [parallel test lanes](../lessons/spawn-lane-operations.md#parallel-lanes-can-collide-on-test-basenames),
  [adapter-suite coverage](../lessons/verification-and-review-discipline.md#run-every-adapter-suite-in-a-gate),
  and [dependency injection](../lessons/harness-integration.md#a-function-local-import-hides-a-layering-cycle).
