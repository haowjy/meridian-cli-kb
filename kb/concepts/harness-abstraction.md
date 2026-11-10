# Harness abstraction

A harness is an external AI runtime (Claude, Codex, Cursor, OpenCode, Pi) integrated through a validated `HarnessBundle`. Launch policy resolves intent; the harness layer translates it into native mechanisms and normalizes events, and never re-runs policy precedence or model routing. Ports and the steps for adding a harness: the source checkout's `src/meridian/lib/harness/AGENTS.md` and `.context/CONTEXT.md`. Adding a harness still touches more than one adapter and its registration; `HARNESS_EXTENSION_TOUCHPOINTS` lists the extra places.

## Seams

- **Connections own live communication; extractors recover session and report data.** `SpawnManager` is the only consumer of a connection's event stream and fans events out ([attempt facts](../architecture/attempt-facts-and-delivery.md)).
- **Approval requests go through a handler.** It separates who decides an approval from how the transport sends it.
- **A managed primary's TUI talks to its backend directly.** Routing the user's input through Meridian would add latency and complexity. Claude primaries use a PTY because Claude has no separate backend to attach to.
- **Recursion is bounded by spawn depth (`MERIDIAN_MAX_DEPTH`).** A per-command launch guard was dropped because it also stopped delegated agents from testing launches.

## Parent and child events

A parent event scope classifies events instead of filtering them: a child's events stay visible to subscribers but can't complete or fail the parent, clear its signals, or become its report. OpenCode's single global event stream makes this boundary matter most.

## A delivery result is not a read receipt

A connection reports whether the harness acknowledged a message or the message was only posted. Neither proves the model read it ([primary messaging](../architecture/primary-messaging.md)).

## Claude spawns use the five-minute cache

Meridian forces a five-minute prompt-cache TTL for non-interactive Claude spawns. Claude treats `claude -p` as a main conversation, so on a subscription a short-lived agent would otherwise write the pricier one-hour cache. The cost is a rebuild after five minutes; no quota saving has been measured. Claude's settings-file environment can still override it ([prompt caching](https://code.claude.com/docs/en/prompt-caching#choose-the-ttl-yourself), [retention research](../research/prompt-cache-retention.md)).

Harness specifics: [Claude sessions](../architecture/claude-native-sessions.md), [Pi sessions](../architecture/pi-native-sessions.md), [Pi runtime](../architecture/pi-runtime.md), [OpenCode](../architecture/opencode-harness.md), [Cursor](../architecture/cursor-harness.md).
