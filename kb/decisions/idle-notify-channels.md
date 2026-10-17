# Decision: Idle Notification Channels

This page owns the notification-side decisions for idle automation: the leaf
package boundary, notice context, channel secrets, and deliberate spawn
notifications. The idle schedule and harness mechanism remain in
[Idle Notifications](idle-notifications.md) and
[Idle Adapter Architecture](../architecture/idle-adapters.md).

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
cache warnings use `[cache warning]`. (Compaction-result titles such as
`[compacted]` went with Meridian's idle compaction,
[D-idle-compaction-removed](idle-notifications.md#d-idle-compaction-removed--meridian-no-longer-compacts-idle-sessions).) The
terminal prefix is the recorded tmux session when present, and is omitted
outside tmux. A work item is labelled explicitly; when there is no work item,
an explicitly labelled project is used when available. Custom titles supplied
to `meridian notify` retain the terminal/actor/work context in the body rather
than dropping it. There is no duplicated tmux footer.

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
- [Config Precedence](../concepts/config-precedence.md): standard precedence and config-key declarations.
- [Launch System](../architecture/launch-system.md): child-environment secret boundaries.
- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md): cache facts behind notice countdowns.
