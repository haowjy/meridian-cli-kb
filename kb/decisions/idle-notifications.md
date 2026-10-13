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
| 60 s | "waiting on you" | yes | no |
| TTL − 15 min | "cache cold in 15m" | yes | yes |
| TTL − 5 min | compact if every guard passes, then report | yes | no |
| user sends a prompt | cancel everything pending | — | — |

Stages that don't fit inside the TTL are dropped. With a 5-minute cache only the
push fits; with an unknown TTL there is only the push. Which harness has which
TTL is in [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md).

## Where it lives

| Concern | Code |
|---|---|
| Notify delivery and CLI | `src/meridian/lib/notify/`, `src/meridian/cli/notify_cmd.py` |
| Idle policy, sidecar, CLI, and state | `src/meridian/lib/idle/`, `src/meridian/cli/idle_cmd.py`, `src/meridian/lib/state/idle_store.py`; TTL detection reads the chat's `native_store` through `src/meridian/lib/state/session_store.py` |
| Harness ports and Python adapters | `src/meridian/lib/harness/idle_types.py`, `bundle.py`, `claude_idle.py`, `pi_idle.py`, `codex_idle.py`, `opencode_idle.py` |
| In-harness adapters | `src/meridian/claude_runtime/meridian-idle/`, `src/meridian/pi_runtime/extensions/meridian-idle/` |
| Launch hosting | `src/meridian/lib/launch/process/primary_attach.py` |
| Layering guard | `tests/unit/harness/test_layering.py` |
| Agent notification rule | meridian-base `skills/work-artifacts/SKILL.md` |

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
  impossible.
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
whether there is a draft, how many tokens are in context). Core owns the
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
heuristics are estimated, and so are explicit, configured and default TTLs
([D-idle-ttl-detection](#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd)). This changes notice context only;
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
4. stretch closed, or the timer is stale
5. timer fired early
6. timer fired late (machine slept)

Compaction only, after those:

7. compaction disabled for this harness
8. cache already cold
9. harness busy
10. unsent draft
11. agents running in the harness
12. child spawns still active
13. context under `min_compact_tokens`
14. harness's own auto-compaction off

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
| Pi | a fourth bundled extension | `-e` entrypoint, interactive launch profile only | `ctx.compact()` | push, warn, compact on `anthropic` and `openai` under long retention, which Meridian sets for primaries ([D-pi-cache-retention](#d-pi-cache-retention--meridian-launches-pi-primaries-with-long-cache-retention)), and on `openai-codex` in either mode (estimated). Push only on `xai`, `deepseek`, any other provider, and `anthropic`/`openai` with `PI_CACHE_RETENTION=short` ([D-idle-ttl-detection](#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd)) |
| OpenCode | task in meridian's attach launcher | optional bundle hook `primary_idle_sensor` | `POST /session/{id}/summarize` with the session's current model | push only (`ttl_seconds` 300) |
| Codex | Codex `notify` command plus a task in the attach launcher | `-c notify=[…]` on the app-server, interactive only | tmux: type `/compact`, verify, press Enter, wait for an appended `compacted` rollout record | push, warn, compact (`ttl_seconds` 1800) |

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
impossible, but it does not remove the tmux dependency from production: the
current actuator is unchanged, approval routing was not exercised, and the
backend does not expose an unsent TUI draft. The pane-based draft guard must
therefore remain conservative, including vetoing when draft state is unknown.
Provenance: `evidence/codex-backend-compact-probe.md` (`work:idle-cache-notify`).

**Why the Claude mod is bundled, not mars-synced:** the mod is code coupled
to the `meridian idle` CLI contract of the same meridian version, not prompt
content. Injecting it with `--plugin-dir` only on interactive launches makes
"primaries only" hold by construction: spawns run `claude -p` and never get
the flag. Claude sessions started outside meridian can opt in through
`CLAUDE_CODE_PLUGIN_DIRS`. (User decision U5.)

### Adapter contracts

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
- **OpenCode.** Summarize keeps the session model when the current provider and
  model are supplied. The session GET times out at 10 s and the summarize POST
  at 300 s; a hung backend reports `failed` instead of wedging later stretches.
  The sensor marks its own compaction before the POST, and the first matching
  `session.idle` ends compaction classification independently of POST response
  ordering. Failed, timed-out, and cancelled summarize requests clear the
  classification in the request's `finally`, so recovery does not require a
  later event. Messages tied to the compaction part remain classified as
  synthetic; the next real user return is not swallowed. Draft text comes
  from the attached TUI pane; context size is unobservable and uses the
  fail-open rule.

**Rejected:**

- **Shipping the mod through `mars sync`:** it would separate the mod's version
  from the CLI it calls.
- **An OpenCode TypeScript plugin on `session.idle`:** it would duplicate the
  event stream meridian already reads, and the launcher can POST summarize
  directly.
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

**Known fragility:** each adapter embeds harness behaviour that can change
upstream. Most drift fails closed: returns go unseen or compaction is skipped.
These cases fail open or fail silently:

- **Codex busy text `Working (`.** If it changes, `busy` is always false and
  the actuator could type `/compact` into a working session. This fails open.
- **Claude `$.config.list()` key `autoCompact`.** If it is renamed, the mod
  compacts for a user who turned auto-compact off in settings. This fails open;
  core still honours `DISABLE_AUTO_COMPACT`.
- **Return-signal renames** (Claude `origin.kind`, Pi `input.source`, Codex
  cumulative `input-messages`). Stretches stop closing, so notifications stop
  after the first stretch. This fails silently.
- The Codex tmux actuator narrows the race with a returning user but can't
  close it.

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
`ttl_seconds = 1800`, OpenCode `300`. Claude reads the transcript TTL, and Pi
maps its provider and `PI_CACHE_RETENTION` to a TTL. Both detectors are covered
in [D-idle-ttl-detection](#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd).

## D-idle-ttl-detection — Detection reads launch records, never a cwd

**Problem it fixed:** idle compaction shipped in PR #550, then fired once in two
days. An unknown TTL makes `timeline.schedule` drop warn and compact, and
detection returned unknown on almost every real session:

- **Claude.** The mod sent `--cwd $.session.cwd()`, and `claude_idle` built
  `~/.claude/projects/<slug(cwd)>/<id>.jsonl` from it. `$.session.cwd()`
  follows every Bash `cd`. Claude files the transcript under the slug of the
  *launch* directory. Any session whose agent changed directory missed its own
  transcript, even though every assistant row recorded `ephemeral_1h` tokens.
- **Pi.** The detector knew only `anthropic` and `openai`. Real primaries use
  `openai-codex`, `xai` and `deepseek`. All ten sampled Pi idle state files had
  `ttl=None`.

**Decision (D1–D2, `chat:c7500`):** core resolves the chat's launch-recorded
`native_store`. `IdleService.arm` reads `MERIDIAN_CHAT_ID` from the session env.
It looks up that chat's record in the project runtime's `sessions.jsonl`, under
`MERIDIAN_PROJECT_DIR`. It then passes the record's absolute `native_store` to
the harness's `detect_ttl`. The `DetectIdleTtl` port carries `native_store`, not
a cwd. Claude resolves `<native_store>/<live session id>.jsonl` exactly, through
the same header-checked `resolve_claude_native_session_file` that resume uses
([Native Session Binding](../architecture/native-session-binding.md)). `--cwd` is
gone from `meridian idle arm`, the service, the port and both mods.

**Rule:** never pass, derive or fall back to a cwd for transcript lookup. An
agent can `cd`, but the transcript stays where the launch filed it. The rule is
the same as for resume: native resolution uses the recorded store. A miss
returns unknown, not a guess:

- no `MERIDIAN_CHAT_ID`;
- no record, a record for another harness, or an empty or relative store;
- a missing file, a header mismatch, or a bad ID.

A wrong TTL cannot happen, because the exact ID must exist in that store.

**Consequences:**

- After `/clear`, the live session ID is joined to the entry chat's store. This
  works because Claude files the successor under the same project-store parent.
  The regression test arms a successor ID against the entry record. If Claude
  ever files successors elsewhere, detection degrades to unknown, not to a wrong
  TTL.
- A Claude session started outside Meridian that opts into the mod through
  `CLAUDE_CODE_PLUGIN_DIRS` has no `MERIDIAN_CHAT_ID`. It now gets push only,
  unless `ttl_seconds` is configured.
- Every arm without a configured TTL parses `sessions.jsonl`: 0.09 s on 13 MB,
  0.19 s on 30 MB. That is accepted because arm runs off the TUI path. Revisit
  if session logs grow tenfold, or move the read onto the SQLite projection.

**Pi provider table (D3–D5).** `pi_idle._PROVIDER_TTLS` maps the provider and
the retention mode to a TTL. No match means unknown.

| Provider | Short retention | Long retention | Why |
|---|---|---|---|
| `anthropic` | 300, known | 3600, known | Pi sends `cache_control.ttl`. |
| `openai` | unknown | 1800, estimated | Pi asks for `24h`, which usually lasts about 30 min. |
| `openai-codex` | 1800, estimated | 1800, estimated | Mirrors the Codex harness default (U4). Pi's Codex adapter ignores `PI_CACHE_RETENTION`. |
| `xai` | unknown | unknown | Best-effort cache that "can be evicted at any time"; no TTL to schedule against. |
| `deepseek` | unknown | unknown | Disk cache that lasts "a few hours to a few days"; there is no pre-expiry point. Compact for context pressure, not cache expiry. |

The `openai-codex` row is a tech-lead call (D5), flagged to the user and not yet
confirmed. Research (`spawn:p7724`) found no published TTL for the ChatGPT
backend. It also refuted mapping the public API's GPT-5.6 30-minute TTL onto that
backend ([research](../research/prompt-cache-retention.md#chatgpt-backend-codex-and-pis-openai-codex)).
The Codex harness already runs on the same backend with the 1800 estimate. So
Pi should match it rather than diverge, and `estimated=true` puts `est.` on its
notices. **Risk:** if the backend cache lives under 25 minutes, the compaction
lands on a cold cache. Revisit if observed cache-hit data disagrees.

**Rejected:**

- **The session's current cwd** (the shipped behaviour): it follows the agent's
  `cd`, so the lookup misses once the session moves. The user chose "where it
  spawned from".
- **Treating Pi `xai` or `deepseek` as known or estimated:** neither provider
  publishes an idle TTL. A guessed TTL would make an estimate look like a
  provider fact.
- **Leaving Pi `openai-codex` unknown**, which the research recommended: it
  would make Pi on the ChatGPT backend the one route where warn and compact
  never fire, while the Codex harness on the same backend schedules both.

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
  ([D-idle-ttl-detection](#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd)).
  *Superseded (2026-10-13):* this page used to keep `openai-codex` push-only,
  reasoning that warning and compaction "would target a cache that isn't
  there". That conflated "no retention knob" with "no cache". The ChatGPT
  backend caches automatically by `prompt_cache_key`, the same way it does for
  the Codex harness. Normalising `openai-*` IDs to `openai` stays withdrawn,
  because Pi does not extend retention there.
- **OpenCode** stays push-only by default. Its 5-minute Anthropic cache is
  hard-coded, with no external switch.

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
| Claude-only `claude_runtime` touches six projections; three tmux helpers | `src/meridian/lib/harness/.context/FUTURE` |
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

## Related

- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md):
  the TTL facts behind the defaults.
- [Config Precedence](../concepts/config-precedence.md): standard precedence
  and how config keys are declared.
- [Launch System](../architecture/launch-system.md): the bind seam, the child
  env boundaries and the primary idle sidecar.
- [Harness Adapters](../codebase/harness-adapters.md#idle-bundle-ports): the
  idle bundle ports.
- GitHub #548: `primary.autocompact` is written into
  `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` as a token count; that variable expects a
  percentage. Found during this design; tracked separately.

## Revisit when

- A harness update invalidates one of the runtime-probed event or compaction
  contracts above, especially a fail-open one.
- A provider changes its default cache retention or the price of long cache
  writes, or Codex/OpenCode add a retention setting.
- Pi applies `PI_CACHE_RETENTION` in its `openai-codex` adapter, or renames or
  drops the variable.
- OpenAI publishes a ChatGPT-backend cache TTL, or observed cache hits after
  idle contradict the 1800 s Codex and Pi `openai-codex` estimate.
- Claude files a `/clear` successor outside the launch store, or Meridian stops
  recording `native_store` on the chat; Claude TTL detection would then return
  unknown.
- A harness gains a native idle or compaction hook that would replace a
  launcher task or the tmux actuator.
- The `arm` contract changes next, or a fifth adapter is about to be written.
  Do #549 first (pending stages from `arm` and `status`), plus the Codex
  identity-pin op from `lib/idle/.context/FUTURE`, so the new adapter
  doesn't become a fifth copy of the protocol.
- A new idle capability needs a bundle port. Group the ports first (#554).

## Sources

- `work:idle-cache-notify`: `design/`, `DIVERGENCE/`, `reviews/`, and `evidence/`
  hold implementation history, rejected mechanisms, and verification records.
- `chat:c7280`: user decisions U1–U6; `chat:c7347`: Pi retention decision U9.
- `work:idle-ttl-detection` (`chat:c7500`, branch `fix/idle-ttl-detection`):
  before-state evidence and D1–D5 in `decisions.md`, Pi provider research
  `research/pi-provider-cache-ttl.md` (`spawn:p7724`), review
  `reviews/review-1.md`, probe `evidence/probe-ttl-detection.md`.
- Process lessons: [parallel test lanes](../lessons/spawn-lane-operations.md#parallel-lanes-can-collide-on-test-basenames),
  [adapter-suite coverage](../lessons/verification-and-review-discipline.md#run-every-adapter-suite-in-a-gate),
  and [dependency injection](../lessons/harness-integration.md#a-function-local-import-hides-a-layering-cycle).
