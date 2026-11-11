# Harness abstraction

A harness is an external AI runtime (Claude, Codex, Cursor, OpenCode, Pi) integrated through a validated `HarnessBundle`. Launch policy resolves intent; the harness layer translates it into native mechanisms and normalizes events, and never re-runs policy precedence or model routing. Ports and the steps for adding a harness: the source checkout's `src/meridian/lib/harness/AGENTS.md` and `.context/CONTEXT.md`. Adding a harness still touches more than one adapter and its registration; `HARNESS_EXTENSION_TOUCHPOINTS` lists the extra places.

## Seams

- **Connections own live communication; extractors recover session and report data.**
- **Raw harness events stay open; the normalized layer is closed.** Unpinned harnesses can add event types, and a closed semantic layer still gives shared code a stable vocabulary.
- **Approval requests go through a handler.** It separates who decides an approval from how the transport sends it.
- **A managed primary's TUI talks to its backend directly.** Routing the user's input through Meridian would add latency and complexity. Claude primaries use a PTY because Claude has no separate backend to attach to.
- **Recursion is bounded by one spawn-depth cap (`MERIDIAN_MAX_DEPTH`) across all commands,** so runaway delegation stops however it recurses. A per-command launch guard was dropped because it also stopped delegated agents from testing launches.

## One consumer per event stream

`SpawnManager` consumes the connection's event stream and fans events out to hooks and subscribers. Claude's stream permits one consumer, so any second reader would compete with the drain loop. Pi's raw `agent_end` stays observable, but its decoded attempt outcome waits for `agent_settled`, because a native retry or compaction can follow `agent_end` ([Pi](pi.md#quiescence)). When a harness changes event dialect, every consumer of the stream follows the resolved transport, not just the main connection ([OpenCode 2.x idle sensor](../spawns/idle-notifications.md#opencode-2x-sensor)).

Attempt facts are folded from the events one turn actually observed, not from a time window over a later transcript; native transcripts serve conversation reads separately ([native transcript reads](../state/native-transcript-reads.md)). If a fold fails, a known harness-specific usage total survives, but a generic partial sum is discarded rather than shown as a total. Unknown usage is not zero. A crashed runner loses its in-memory fold, so recovery relies on durable report and lifecycle evidence.

## Parent and child events

A parent event scope classifies events instead of filtering them: a child's events stay visible to subscribers but can't complete or fail the parent, clear its signals, or become its report. OpenCode's single global event stream makes this boundary matter most.

## A delivery result is not a read receipt

A connection reports whether the harness acknowledged a message or the message was only posted. Neither proves the model read it ([messaging](../spawns/messaging.md)).

## Upstream behavior drifts

Adapters depend on upstream event names, payloads and binaries, which change without notice. A sensor that never fires needs positive evidence of activity, not just an absence of errors, and should be probed on the version users actually run. Before removing a workaround for an upstream bug, reproduce the original failure on the current binary: a closed issue doesn't show it was fixed. A running harness keeps the mod or extension it loaded at session start after the CLI on `PATH` is upgraded, so changing a flag or reply shape that bundled code calls can break live sessions until they restart.

Harness specifics: [Claude](claude.md), [Pi](pi.md), [OpenCode](opencode.md), [Cursor](cursor.md).
