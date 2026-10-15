# Decisions: Idle Notifications and Idle Compaction

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
| 60 s after a turn of 120 s or more; 600 s after a quicker or unknown-length turn | "waiting on you" | yes | no |
| TTL − 15 min | "cache cold in 15m" | yes | yes |
| TTL − 5 min | compact if every guard passes, then report | yes | no |
| user sends a prompt | cancel everything pending | — | — |

The push delay comes from `[idle] push_seconds`, `long_turn_seconds` and
`quick_turn_push_seconds` (since v0.9.3), so a one-line answer doesn't page the
user a minute later. Stages that don't fit inside the TTL are dropped. With a
5-minute cache only the push fits; with an unknown TTL there is only the push.
Which harness has which TTL is in
[Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md); what
each harness actually schedules today is in
[Per-harness coverage](../architecture/idle-adapters.md#per-harness-coverage).

## Invariants to keep

- **Compaction checks live only in the ordered guards.** `lib/idle/guards.py`
  is one ordered pure function that matches the guard list below line by line.
  A new check goes there, never into an adapter. Adapters keep only their
  last-second veto re-checks between `act` and the compact call.
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
  on `lib/idle` ([D-idle-leaf-packages](#d-idle-leaf-packages--libnotify-and-libidle-are-leaf-services)).
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

**Decision:** Each harness adapter only observes and acts. It reports
facts to core (a turn finished, the user typed a prompt, a timer fired,
whether the harness is busy, how many tokens are in context). Core owns the
timeline, every guard, the state file, config resolution and notification
delivery. For compaction, core replies `act` or `skip`. An adapter never reads
meridian config, never writes idle state, never sends a notification itself,
and never compacts without `act`.

In-session TypeScript adapters (Claude, Pi) talk to core through a hidden
`meridian idle config|status|arm|return|fire|done` CLI and pass `--interactive`
on **every** call. Codex's native `notify` callback enters through
`meridian idle event`. An absorbed arm includes its `reason` on the real wire,
so adapters can distinguish preserved timers from a genuine reschedule.
Adapters hosted in meridian's launcher (Codex, OpenCode) call the same service
in-process. Facts visible in env reach core through
harness bundle hooks, so the idle package never names a harness:
`DISABLE_AUTO_COMPACT` (Claude) and `OPENCODE_DISABLE_AUTOCOMPACT` through the
`autocompact_off` hook, and `PI_CACHE_RETENTION` through Pi's `detect_ttl`.
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

## D-idle-leaf-packages — `lib/notify` and `lib/idle` are leaf services

**Decision:** Two new packages, modelled on `lib/artifact/` (a self-contained
feature whose CLI is its only entry point), not on `lib/ops/`.

- **`lib/notify/`** delivers notices. It builds the reason-first session
  context (terminal, actor, and explicit work/project label; see
  [D-idle-notice-context](#d-idle-notice-context--reason-first-titles-and-send-time-cache-countdown))
  and fans out to a channel registry: `ntfy`, `smtp`, `gmail` (an SMTP
  preset), `command`, `none`. A new channel is one module plus one registry
  entry. `meridian notify` works with idle turned off. ntfy titles are RFC
  2047-encoded, because the HTTP `Title` header is Latin-1 and the label's `·`
  would otherwise arrive mangled.
- **`lib/idle/`** holds the policy: a pure timeline, pure guards, a service
  over the state file, and the sidecar loop. It builds `lib.notify.Notice`
  values and calls `lib.notify.send` directly; `lib/notify` never imports
  `lib/idle`. The service's `notify_sender` parameter is only a seam for
  tests.
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
cache warnings use `[cache warning]`, and compaction results use their result
reason (`[compacted]`, `[compaction failed]`, or `[compaction skipped]`). The
terminal prefix is the recorded tmux session when present, and is omitted
outside tmux. A work item is labelled explicitly; when there is no work item,
an explicitly labelled project is used when available. Custom titles supplied
to `meridian notify` retain the terminal/actor/work context in the body rather
than dropping it. There is no duplicated tmux footer.

Idle and cache-warning notices calculate the remaining cache lifetime **at send
time**, from the persisted idle anchor and TTL. A known TTL is shown plainly;
an estimated TTL is prefixed `est.`; an unknown TTL has no countdown. The
renderer uses `<1 min until cold` below a minute and `cache cold` after the
deadline. Compaction-result notices omit the old countdown because compaction
may refresh the cache. `ttl_estimated` is persisted with the idle state, with
old records defaulting conservatively to estimated. Claude's observed transcript
TTL and Pi's Anthropic retention are known. Pi's `openai` and `openai-codex`
heuristics and OpenCode's `openai` mapping are estimated, and so are explicit,
configured and default TTLs
([D-idle-ttl-detection](idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd)). This changes notice context only;
the schedule and compaction guards are unchanged.

**Why:** A reason-first title tells the user why a notice arrived, while the
terminal and explicit work label identify which session needs attention. Rendering
from persisted state at send time avoids publishing a stale precomputed deadline.
The confidence marker prevents a provider heuristic or configured fallback from
looking like a provider guarantee.

## D-idle-stretch-at-most-once — Each stage fires at most once per stretch

**Decision:** A *stretch* runs from one user prompt to the next. Push, warn
and compact each happen at most once per stretch. A compaction counts once it
is attempted, whatever the result. Each finished turn inside the stretch
re-anchors the schedule, because the cache was just refreshed. Re-anchoring
moves pending stages later, but never re-enables a stage that already ran. An
arm on a closed stretch always opens the next one, even after a successful
compaction.

From the moment core says `act`, plain turn-finished reports are ignored so the
compaction's own turn can't restart the timeline. Only `done ok` extends that
*compaction window* by 30 seconds. `done failed|vetoed` clears the window and
the expected-compaction-turn marker, so the next real arm is not swallowed. A
user return is always honoured, even inside the window. That covers a
positively identified prompt and an arm that carries `--implies-return`
(Codex's only return signal, because Codex's own compaction fires no notify).

The guard order puts the cheap checks first and the compaction-only checks
last:

1. not a primary
2. idle disabled for this harness
3. stage already done in this stretch
4. stretch closed
5. timer is stale (the anchor moved)
6. stage not scheduled
7. timer fired early
8. timer fired late (machine slept)

Compaction only, after those:

9. compaction disabled for this harness
10. cache already cold
11. harness busy
12. agents running in the harness
13. child spawns still active
14. compaction not worth it: growth below the per-harness break-even, or
    context or baseline unknown
    ([D-idle-compaction-growth-floor](idle-compaction-growth-floor.md#d-idle-compaction-growth-floor--compact-on-growth-past-a-per-harness-break-even))
15. harness's own auto-compaction off

There is no draft guard: unsent composer text never blocks compaction
([D-idle-compact-over-drafts](#d-idle-compact-over-drafts--compaction-wins-over-unsent-text)).

**Unknown facts:** an unknown context or baseline fails closed
(`skipped:context-unknown`, `skipped:baseline-unknown`). The original rule (D6)
let unknown context compact, on the assumption that compacting a small context
costs little. Measured costs showed that a compaction can cost more than it
saves, so v0.10.6 made unknown skip.

**State:** one JSON file per harness session at
`~/.meridian/idle/<harness>-<native_session_id>.json`, written atomically and
read truncation-tolerantly, with lazy deletion after 7 days. Timers live in
memory wherever the adapter runs and are re-derived from the file after a reload.

**Why:** A Claude mod hot reload and a Pi `/reload` both wipe in-memory state;
without a durable marker a reload re-sends the push and can compact twice.
The file is the only record (files as authority); timers are disposable
(crash-only).

**Compaction size check:** growth since a baseline against a per-harness
break-even, not total context
([D-idle-compaction-growth-floor](idle-compaction-growth-floor.md#d-idle-compaction-growth-floor--compact-on-growth-past-a-per-harness-break-even)).

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
`stretch_open = false`, `returned_at_ms = now`, compaction window and
expected-compaction flag cleared). Claude and Pi then find nothing open to
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
  returns silently. [D-spawn-run-identity](state.md#d-spawn-run-identity)
  later decided that it does; that reopen is built but not merged. It also added about five concepts (owner identity, a
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

## D-idle-compact-over-drafts — Compaction wins over unsent text

**Status:** decided by the user (2026-10-15); merged in PR #583 (`8bd2f4775`),
released in v0.10.5. `work:idle-compaction-floor` ("User: compact even with a
draft"), evidence `evidence/drop-draft-guard.md`.

**User rule:** "I would rather we compact than keep any drafts." The user said
this after asking what happens when a tmux pane is mangled or resized.

**Problem:** compaction used to skip when the composer held unsent text, and
also when the adapter could not tell. Only Claude and Pi could see a draft
through their host APIs. Codex and OpenCode guessed from the tmux pane. The
OpenCode 2.x pane parser looked for a footer separator that 2.0.28 never
renders, so every 2.x compaction skipped as `draft-unknown`. Its replacement, a
parser anchored on the composer border, still depended on a pane that resizes
and redraws. A rare lost draft costs less than a guard that silently disables
compaction.

**Decision:** delete the draft fact and its guard everywhere: the core guard,
`IdleFacts`/`GuardFacts`, the CLI `--draft` flag, the OpenCode pane parser, and
the Claude and Pi draft vetoes. Claude, Pi and OpenCode compact through APIs
that do not touch the composer. Codex still compacts by typing into its TUI,
so it has to clear the composer first. The busy check, the `/compact`
verify-before-Enter step and the TUI-alive check stay.

**Codex clears the composer with a screen-gated Ctrl+C.** Codex 0.162.1 has
no single key that clears a multi-line composer: Ctrl+U deletes only to the
start of the line. Ctrl+C clears the whole composer, including attachments,
but only when the composer is non-empty. On an empty composer the key falls
through to the outer handler, which interrupts running work or quits Codex.
The double-press quit confirmation is disabled in this release. So the
sequence is:

1. Capture the pane and require the focused composer layout: two blank rows
   above a column-zero `›` prompt, indented continuation rows, the blank
   bottom margin, an indented status row, and the footer (`… for shortcuts`
   when empty, blank with a draft). Reject `Working (`.
2. Type the primer `ix`, which makes the composer non-empty in both normal and
   Vim mode (`i` enters insert mode). Confirm that the visible text is exactly
   the previous text plus `ix`.
3. Re-check busy and liveness, then send Ctrl+C.
4. Confirm the composer shows empty, type `/compact`, verify it, press Enter,
   and wait for a new `compacted` rollout record.

A slash-command popup takes the blank rows above the prompt. Modals, the
agents overview and transcript views replace the footer. None of them match
the layout, so Ctrl+C is never sent to them. The first version without the
layout gate was sent back in review. In that version a key could reach a modal
or the agents overview, and an unprimed Ctrl+C could quit Codex or interrupt
it. The deleted empty-`›` draft check had been blocking those screens by
accident.

**An unreadable screen retries, then notifies.** If the layout doesn't match,
the actuator re-captures for about 10 s (three reads). If it still can't
recognize the screen, it vetoes with `screen-not-recognized`, the same reason
it uses when the primer or the cleared composer fails to verify. No key is
sent. Core recognizes that one reason and sends a push-only notice:
"Codex not compacted: Meridian couldn't read its screen", plus the minutes of
cache left. The decision about who to notify stays in the idle policy layer,
not the adapter (user decision). Other vetoes and failures keep the generic
compaction notice.

**Verified live** (codex-cli 0.162.1, `gpt-5.6-luna`, scratch homes):

- A three-line draft was cleared and compacted, and Codex kept running.
- An open `/` popup gave `composer-not-focused` before any key was sent.
- Five resizes just before compaction still compacted cleanly.
- A popup left open gave the `screen-not-recognized` veto after 10.02 s, with
  exactly one notice.

**Accepted residual risk:** the pane can change between the capture and the
keys. No tmux key sequence is atomic with the TUI's handling. The primer makes
the quit path unreachable in a normal composer, and the exact `/compact` check
blocks submission if stray text remains.

**Rejected:**

- **Keeping the guard with a better OpenCode pane parser.** A parser was built
  (`spawn:p7902`) and then deleted unmerged. It still depended on the pane, and
  the user preferred compaction.
- **Silent vetoes for an unreadable Codex screen.** Compaction would stop
  without the user knowing. One notice per stretch costs little.

**Related, not decided here:** Codex and OpenCode TUIs resize correctly when
run natively, but their layout gets mangled under Meridian primaries. That is
tracked in `work:primary-resize-mangling`.

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
per call, not only on `config`: core checks the role on every `arm`, `fire` and
`done`. When only `config` carried it, an outside-meridian Claude session read
`enabled: true` and then had every arm declined (G2 C1).

**Caveat:** an agent inside a primary that runs a bare `claude` inherits
`primary`. The Claude mod therefore also requires an interactive TUI surface
before it arms.

## D-idle-adapter-hosting — Where each adapter runs

**Decision:**

| Harness | Adapter host | How it gets there | Compacts with | Default |
|---|---|---|---|---|
| Claude | mod inside Claude Code | bundled in meridian-cli, `--plugin-dir` on interactive launches only | `$.session.compact()` | push, warn, compact |
| Pi | a fourth bundled extension | `-e` entrypoint, interactive launch profile only | `ctx.compact()` | push, warn, compact on `anthropic` and `openai` under long retention, which Meridian sets for primaries ([D-pi-cache-retention](#d-pi-cache-retention--meridian-launches-pi-primaries-with-long-cache-retention)), and on `openai-codex` in either mode (estimated). Push only on `xai`, `deepseek`, any other provider, and `anthropic`/`openai` with `PI_CACHE_RETENTION=short` ([D-idle-ttl-detection](idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd)) |
| OpenCode | task in meridian's attach launcher | optional bundle hook `primary_idle_sensor`; the sensor dialect follows the connection's resolved backend version ([D-opencode-idle-v2](#d-opencode-idle-v2--opencode-2x-gets-its-own-sensor-chosen-by-the-resolved-transport)) | 2.x: `POST /api/session/{id}/compact`, completion tied to the returned inbox ID. 1.x: `POST /session/{id}/summarize` with the session's current model | 2.x `openai` provider: push, warn, compact (1800 s, estimated). Other providers and 1.x: push only. No `ttl_seconds` default ([D-idle-ttl-detection](idle-ttl-detection.md#opencode-provider-ttl)) |
| Codex | Codex `notify` command plus a task in the attach launcher | `-c notify=[…]` on the app-server, interactive only | tmux: screen-gated composer clear, then type `/compact`, verify, press Enter, wait for an appended `compacted` rollout record ([D-idle-compact-over-drafts](#d-idle-compact-over-drafts--compaction-wins-over-unsent-text)) | push, warn, compact (`ttl_seconds` 1800) |
| Cursor | none | no primary path | — | no idle |

The Default column describes what gets scheduled. When the compact stage
fires, Claude, Codex and OpenCode 2.x `openai` compact only after enough growth.
Pi's stage always skips as `not-worth-it` unless `compact_min_growth_tokens`
is set
([D-idle-compaction-growth-floor](idle-compaction-growth-floor.md#d-idle-compaction-growth-floor--compact-on-growth-past-a-per-harness-break-even)).

The launcher task is an asyncio task inside `PrimaryAttachLauncher.run`. It
starts after the TUI is running and follows the TUI launch task's liveness, not
the observer stream. It is cancelled and awaited before the connection is
stopped. Synchronous policy and store calls run through `asyncio.to_thread`.
Compaction runs as its own task, so a user return doesn't cancel it mid-action
or lose its `done`. Failures in raw events, the event stream, timers, store
polling and compaction are contained per event and written to spawn-local debug
telemetry, never to the TUI's terminal.

**Why the launcher, not plugins, for Codex and OpenCode:** both are
managed-attach primaries, so a meridian process already lives for the whole
session, runs an event loop and holds the backend connection. OpenCode needs
no plugin at all: meridian already consumes its raw live events and can POST to
its server. In the current Codex attach path, the TUI/observer arrangement
leaves Meridian dependent on `notify` for the turn sensor, and production's
compaction actuator remains tmux-based. (User decision U5 / design Q4.)

**Bounded Codex backend probe (not a production change):** On `codex-cli
0.162.1`, a second initialized WebSocket client resumed a thread while the
native TUI was attached and successfully called `thread/compact/start`. Both
WebSocket clients and the TUI remained usable, and the TUI rendered the
completion. This corrects any categorical claim that backend compaction is
impossible, but production still compacts through tmux: approval routing was
not exercised, and the actuator was never switched. A backend actuator would
remove the screen-gated composer clear
([D-idle-compact-over-drafts](#d-idle-compact-over-drafts--compaction-wins-over-unsent-text)),
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

The current per-harness coverage, adapter contracts and known fragility are in
[Idle Adapter Architecture](../architecture/idle-adapters.md).

**Rejected:**

- **Shipping the mod through `mars sync`:** it would separate the mod's version
  from the CLI it calls.
- **An OpenCode TypeScript plugin on `session.idle`:** it would duplicate the
  event stream meridian already reads, and the launcher can POST the
  compaction directly.
- **A thread from `runner.py` in the PTY path:** a first draft did this; review
  found it needed cross-thread plumbing.
- **Codex hooks as the return sensor:** they need an interactive trust screen.
- **A spawn → session index for Codex:** meridian's recorded
  `harness_session_id` already is the main thread id.
- **A tmux-pane-activity fallback (D5):** not built, because every harness has
  a better sensor.
- **Idle support for the black-box fallback** (managed attach failed): none,
  accepted as a gap.
- **Sharing adapter code between the Claude mod and the Pi extension:** the
  host APIs differ (`$.process.run` against `node:child_process`). The
  duplicated reply handling is better removed by having core return the
  pending stages from `arm` and `status` (GitHub #549; see
  [Structural debt](#structural-debt)).

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
`meridian idle config --harness <h>` takes precedence over `_MERIDIAN_HARNESS`
when choosing which harness's settings to resolve.

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
  `PI_CACHE_RETENTION`. Idle on that provider still schedules warn and compact,
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
  `openai` sessions schedule warn and compact from an estimated 1800 s
  ([OpenCode provider TTL](idle-ttl-detection.md#opencode-provider-ttl)).

**Rejected:**

- **Leaving Pi on short retention** and documenting `PI_CACHE_RETENTION=long`
  as an opt-in: cheaper per write, but Pi primaries would stay the one harness
  where only the push fires.
- **A separate `apply_env_defaults(child_env, *, run, config)` hook**
  (`2c7c7f78`): review rejected it as a second seam for one concept. Folding
  `run` into `env_defaults` keeps one place where adapter defaults live and one
  missing-or-blank rule in `build_harness_child_env` (`bac44c01`).

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
long work done, or a failure; not for routine progress. Idle push, warn and
compaction run only in primaries.

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

**A failed compaction on OpenCode 2.x has not been captured live.** Successful
Meridian-initiated 2.x compactions have run live: two on 2026-10-14
(`spawn:p7902`), then one through the reconnecting stream (`spawn:p7914`). A
compaction gated by the break-even rule on the `openai` route has not fired
live yet.

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

- A harness update invalidates one of the runtime-probed event or compaction
  contracts in [Idle Adapter Architecture](../architecture/idle-adapters.md#adapter-contracts), especially a fail-open one. Re-run the turn-end probe on
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
- A harness gains a native idle or compaction hook that would replace a
  launcher task or the tmux actuator.
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
- Codex changes its composer layout, its Ctrl+C clear behaviour or its quit
  confirmation. A layout change makes every Codex compaction veto with
  `screen-not-recognized` (loud, one notice per stretch); a Ctrl+C change can
  quit or interrupt Codex
  ([D-idle-compact-over-drafts](#d-idle-compact-over-drafts--compaction-wins-over-unsent-text)).

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
