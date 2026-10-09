# Decisions: Idle Notifications and Idle Compaction

**Status (2026-10-08): built, gated, PR pending.** The core, all four
adapters, the prompt rule and the docs are on `feat/idle-cache-notify`. Three
review gates passed, and so did a cross-harness live smoke. Provenance:
`work:idle-cache-notify` (`design/`, `DIVERGENCE/summary.md`, `reviews/`),
`spawn:p7415` (design), `chat:c7280` (user decisions U1–U6), `chat:c7275`
(planning).

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
| Idle policy, sidecar, CLI, and state | `src/meridian/lib/idle/`, `src/meridian/cli/idle_cmd.py`, `src/meridian/lib/state/idle_store.py` |
| Harness ports and Python adapters | `src/meridian/lib/harness/idle_types.py`, `bundle.py`, `claude_idle.py`, `pi_idle.py`, `codex_idle.py`, `opencode_idle.py` |
| In-harness adapters | `src/meridian/claude_runtime/meridian-idle/`, `src/meridian/pi_runtime/extensions/meridian-idle/` |
| Launch hosting | `src/meridian/lib/launch/process/primary_attach.py` |
| Layering guard | `tests/unit/harness/test_layering.py` |
| Agent notification rule | meridian-base `skills/work-artifacts/SKILL.md` |

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
`meridian idle event`. Adapters hosted in meridian's launcher (Codex, OpenCode)
call the same service in-process. Facts visible in env, such as
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
  with idle turned off. ntfy titles are RFC 2047-encoded, because the HTTP
  `Title` header is Latin-1 and the label's `·` would otherwise arrive mangled.
- **`lib/idle/`** holds the policy: a pure timeline, pure guards, a service
  over the state file, and the sidecar loop. It reaches notification delivery
  through a `NotifySender` protocol with a lazy default, never the reverse.
- **`lib/state/idle_store.py`** owns the state file, because `lib/state/`
  owns all disk I/O.

`lib/idle` depends on config, state, platform helpers, and the neutral
contracts in `lib/harness/idle_types.py`. **`lib/harness` never imports
`lib/idle`.** An AST test (`test_layering.py`) enforces this, including
function-local imports. Harness modules supply sensors, event parsers, env
facts and callbacks through optional `HarnessBundle` ports. When a harness
parser needs stored state, the caller injects it. `meridian idle event` passes
the Codex parser a `session_reader` built from its own `IdleService`, applies
the parsed event, and then calls the bundle's `idle_event_applied` hook. That
hook chains the user's own Codex `notify` command through `atexit`.

**Why:** `lib/ops/` is policy over spawns, sessions and work and drives launch.
Sending a message touches none of that. Keeping notify separate from idle lets
agents call `meridian notify` without the idle machinery. The harness → idle
ban keeps the dependency one-way. Without it, a harness parser that reads
policy state creates a cycle, and a function-local import hides that cycle
from import-time checks (G2 K3).

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
| Pi | a fourth bundled extension | `-e` entrypoint, interactive launch profile only | `ctx.compact()` | push only (5 min cache); long retention enables later stages when the provider is recognized |
| OpenCode | task in meridian's attach launcher | optional bundle hook `primary_idle_sensor` | `POST /session/{id}/summarize` with the session's current model | push only (`ttl_seconds` 300) |
| Codex | Codex `notify` command plus a task in the attach launcher | `-c notify=[…]` on the app-server, interactive only | tmux: type `/compact`, verify, press Enter, wait for the completion marker | push, warn, compact (`ttl_seconds` 1800) |

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
its server. Codex's TUI takes over the observer stream after attach, which
leaves meridian blind to its turns, so Codex additionally needs `notify` as a
sensor. (User decision U5 / design Q4.)

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
  (G2 K2). The actuator types `/compact`, verifies the text, re-checks that the
  TUI is alive, presses Enter, and reports `ok` only when the `Context
  compacted` marker count grows. An empty composer appears as soon as any turn
  starts and is no evidence of completion (G2 K1).
- **OpenCode.** Summarize keeps the session model when the current provider and
  model are supplied. The session GET times out at 10 s and the summarize POST
  at 300 s; a hung backend reports `failed` instead of wedging later stretches
  (G2 O1). The sensor flags its own compaction before the POST, and treats any
  user message created while the flag is set as compaction rather than a
  return. This holds whatever order OpenCode emits `busy` and the
  `{type: "compaction"}` part (G2 O2). Draft text comes from the attached TUI
  pane; context size is unobservable and uses the fail-open rule.

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
  duplicated arm-reply handling is better removed by having core's `arm` return
  only the live deadlines (G2 X3, deferred).

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
- Pi provider IDs other than native `openai` and `anthropic` are treated as
  unknown by TTL detection. For example, `openai-codex` stays push-only even
  with `PI_CACHE_RETENTION=long` until provider-family normalization is added.

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
- **ntfy's email forwarding as an email channel:** not built. It would need a
  role-aware result contract.

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

**Implementation order:** probes for each harness and four enabling refactors
ran first:

- **R1:** catalog-driven loading of new config tables, plus the per-harness model
  factory.
- **R2:** the session role variable.
- **R3:** the launcher idle-task hook.
- **R4:** the Claude bundled-mod seam.

After those came notify core, idle core, the `meridian idle` CLI, the four
adapters, the prompt rule, then docs and a cross-harness smoke test. A review
gate sat between each wave.

## What the plan got wrong

The intent never changed: primaries only, four harnesses, two layers,
everything configurable with standard precedence, and agents can ping on
purpose. These mechanisms did change between the user-confirmed plan, the
accepted design, and the build. Each row is a rejected alternative with the
evidence that killed it.

| Planned | Built | Forced by |
|---|---|---|
| Pi caches for 1 h by default | 5 min; push-only unless `PI_CACHE_RETENTION=long` | Pi README; cache research |
| OpenCode adapter as a TS plugin on `session.idle`, compaction unconfirmed | Python sensor in the attach launcher; summarize confirmed; push-only at 300 s | probe P0c |
| Codex TTL 3600 | 1800 | GPT-5.6+ 30-min retention; Codex sets none (U4) |
| Claude return = `turn.start` with text | composer/bridge `prompt.submit` / `command.run`; compact only from a timer callback; `isInteractive` gate | probe P0a |
| Codex return = non-empty `input-messages`; hooks as fallback | growth of the cumulative count; no hooks; pin on `harness_session_id` | probe P0b |
| Pi extensions already load primary-only | the resolver never checked `interactive`; `meridian-idle` gained the gate | probe P0d |
| SMTP password via env | 0600 file first; env as a leaky fallback | `MERIDIAN_SECRET_*` stripping (U3) |
| Long config key names | `push_seconds`, `warn_minutes`, `compact_minutes` | D3 |
| `--implies-return` honoured only after the window; failed/vetoed `done` opens a window | implies-return honoured any time; only `done ok` opens the window | G1 S2 |
| Role gate = `MERIDIAN_SESSION_ROLE == primary` | unset role + `--interactive` on every call also counts; `spawn` always disabled | G1 S1, G2 C1 |
| `lib/idle` imports `lib/notify` directly | `NotifySender` protocol with a lazy default | the two cores were built in parallel |
| tmux-pane fallback, off by default | not built | D5 |
| ntfy as an email backend | not built | needs a role-aware result contract |

## Review gates

Each gate reviewed a merged tree and ran ruff, pyright, the full pytest suite
and, from G2 on, the TypeScript suites. Every finding was fixed with a
failing-then-passing test before the next wave started.

| Gate | Reviewed | What it changed |
|---|---|---|
| **G0** | R1–R4 refactors | The idle task is cancelled before the connection stops on every teardown path. `tui_alive` follows the TUI launch task instead of the launcher. Claude's generated plugin types no longer ship in the wheel. Per-harness idle fields come from one shared base, with per-harness defaults in the generated models. |
| **G1** | notify core and idle core | The sidecar keeps a compaction alive across a user return, so its `done` is recorded. Core gained Codex's `implies_return` and input-count path. The role gate gained `interactive` from end to end. Failed or vetoed compaction no longer opens a window. ntfy titles became RFC 2047. Sidecar calls moved off the loop, and each event became its own failure domain. |
| **G2** | the four adapters | Every adapter call carries `--interactive` (C1). Pi stopped blocking input and re-checks before compacting (P1, P2). OpenCode got timeouts and marks its own compaction (O1, O2). Codex reports `ok` only on the completion marker, stays quiet while busy, and re-checks the TUI before Enter (K1, K2, K4). `lib/harness` stopped importing `lib/idle`, with an AST test (K3). Pi's Vitest suites run in CI; `claude plugin test` is a required local gate because CI has no `claude` binary (X1). |

Between G1 and G2, a separate fix lane made a closed stretch always reopen on
arm. Without that, a session that compacted and then saw the user return never
pushed again. Process lessons from these gates are in
[Spawn Lane Operations](../lessons/spawn-lane-operations.md#parallel-lanes-can-collide-on-test-basenames)
and [Verification and Review Discipline](../lessons/verification-and-review-discipline.md#run-every-adapter-suite-in-a-gate).

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
- A provider changes its default cache retention, or Codex/OpenCode add a
  retention setting.
- A harness gains a native idle or compaction hook that would replace a
  launcher task or the tmux actuator.
- The `arm` contract changes next. That is when core should return only live
  deadlines and give Codex a dedicated identity-pin call. Both follow-ups are
  tracked in `src/meridian/lib/idle/.context/FUTURE`; the Pi provider
  normalization is in `src/meridian/lib/harness/.context/TODO`.
