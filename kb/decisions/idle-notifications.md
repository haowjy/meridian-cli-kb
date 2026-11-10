# Decisions: Idle Notifications

## Adapters report facts, core decides

Adapters report facts; core decides scheduling and guards. Separate policy in each adapter would duplicate precedence and at-most-once handling across harnesses.

## Each stage fires at most once per stretch

Each push and warning fires at most once between user prompts. Finished turns move pending deadlines but do not re-enable sent stages. Durable state lets adapter reloads reconstruct timers without sending duplicates.

## A Meridian reopen is a user return

Reopening a primary closes the previous idle stretch before the harness starts: the user has returned, so its old push and warning are stale. Reloads do not run the launcher and can reconstruct pending timers. Exit hooks alone cannot cover killed terminals. An in-TUI `/resume` or a Claude TUI started outside Meridian has no launcher and can still adopt the old stretch.

## One `MERIDIAN_SESSION_ROLE` at the bind seam

Idle automation is primary-only. Launch sets `MERIDIAN_SESSION_ROLE` at the shared bind seam so a child cannot inherit its parent's role. A bare Claude launched inside a primary can inherit `primary`; the Claude mod also requires an interactive TUI.

## Where each adapter runs

Claude and Pi use bundled in-harness extensions; managed Codex and OpenCode primaries use launcher tasks. See [idle adapter architecture](../architecture/idle-adapters.md). A running session retains the extension version loaded at startup even after the CLI is upgraded, so an incompatible CLI contract can silently break idle until restart. Bundling the extension with the CLI keeps their versions coupled at launch.

## Separate `[notify]` / `[idle]` namespaces, standard precedence

`[notify]` and `[idle]` use standard config precedence; per-harness settings are more specific within the same level. See `docs/configuration.md` and [config precedence](../concepts/config-precedence.md).

## Meridian launches Pi primaries with long cache retention

Interactive Pi primaries default to `PI_CACHE_RETENTION=long`, unless the user supplies a value, so more sessions can receive a cache warning. Long Anthropic cache writes cost more; `short` opts out. Pi's `openai-codex` adapter did not apply this setting in the observed version. See [cache retention research](../research/prompt-cache-retention.md).

## The push waits for work that will wake the session; the warning does not

A certain background wake holds the waiting push; uncertain work delays it. A known wake before expiry also makes a cache warning unnecessary. The response stays owed until the push is actually sent or the user returns, so a wake before delivery does not incorrectly reset the delay. A false certain classification can suppress a needed push forever: a foreground `meridian spawn wait` can wake the session, while detached work, loops and commands followed by further work do not establish that. See `lib/idle/wake.py` and [idle adapter architecture](../architecture/idle-adapters.md#running-work-and-the-push).

The Claude mod may fire a due push while a new main turn is busy; this gap is tracked in `lib/idle/.context/FUTURE`.

## Only work that wakes the session holds the push

Adapters report work only if its completion wakes the session. OpenCode 2.x releases a hold during a child permission request because that child needs the user; reconnects recheck work because the live stream can miss a completion. Codex reported no wake in two managed-path probes, although a contrary real-use observation remains unexplained; it reports no running work until reproduced. OpenCode 1.x wake behavior is unverified.

## Forward Claude's still-waiting signal without rebuilding prompt state

Claude's still-waiting permission notification is forwarded as a stateless hook event. Reconstructing prompt state by correlating tool events failed when hooks rewrote input, a long approved command remained active, or an agent asked twice. The notice follows idle enablement and does not change the stretch.

## Meridian no longer compacts idle sessions

Meridian does not compact idle sessions; harness-native compaction settings remain independent. A cold return costs a turn, whereas compaction discards transcript detail; session mining can recover older context. The idle service sends a waiting push and cache warning only.

## Verification gaps

The Gmail transport was tested with a fake, but no real email delivery was verified.

## Deferred work

[Session browse crash promotion](https://github.com/haowjy/meridian-cli/issues/558) remains deferred. Its requested 24-hour expiry starts when the crash is first shown, not at detection or indexing, and later views must not reset it.
