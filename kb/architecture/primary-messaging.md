# Messaging a live session

`meridian spawn message` reaches every live session, primaries included, through the same control socket spawned sessions serve. Current behavior: the source checkout's [message route contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/harness/.context/CONTEXT.md#primary-message-route), [framing contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/cli/.context/CONTEXT.md) and `docs/harness-integration.md`. Harness evidence: [reaching a live TUI](../research/reaching-a-live-tui.md). Remaining edges: [#916](https://github.com/haowjy/meridian-cli/issues/916).

## One socket for every primary

The launcher serves a common socket instead of a CLI branch and reachability check per harness. The socket is bound once a route actually exists, rather than predicting support from launch flags, version or an outstanding trust prompt; it can still be stale or refuse. A sender may wait for the route to appear before failing. Advertising a reply route with no usable inbox would offer a command that can't deliver. Older primaries need relaunching to expose the route.

- **Codex and OpenCode:** the observer that watches the backend is also the message route, so it must survive idle silence. A busy Codex primary is steered; OpenCode counts injected turns as activity. Meridian doesn't answer approvals raised by injected turns; they reach the human's TUI.
- **Pi:** an extension serves the socket, because Pi has no external delivery channel. A launcher-to-extension socket would add a second transport, and a file drop couldn't report failure; the cost is maintaining the wire handling in TypeScript.
- **Claude:** Claude's cross-session inbox, below.

## Claude primaries use the inbox

The inbox delivers between tool calls, needs no extra listener, and doesn't depend on a mod loading silently. A mod also can't replace the child's stdin stream: plain `claude -p` exits after going idle even if a mod listener remains, and `$.prompt.submit` waits for the turn to end ([probes](../research/claude-code-mods-and-inbox.md)). The inbox schema comes from the Claude binary and can change without notice, and the inbox gives no receipt that separates held, refused and delivered. Claude adds its peer-safety paragraph to injected messages.

Meridian doesn't send Claude's messaging token, forge a bypass-mode envelope, or override `crossSessionInbound`: each would claim authority the user hasn't given Meridian. A user can set `crossSessionInbound: "accept"` themselves.

> [!FLAG] **Needs human review**: whether to replace the inbox with a mod, and whether bypass-mode primaries should hold injected messages for approval, are undecided. Flagged 2026-11-07.

## The frame carries a reply address

Sender identity travels in the message text, because every route carries text, unlike Claude's peer envelope. The reply goes to the sender's current chat, not its spawn ID or launching chat: continuation changes the spawn ID and a session can move to another chat. A reply command is offered only when a socket file exists, which shows reachability, not that a send will succeed.

Chat and spawn IDs are project-scoped, so a cross-project address includes the project root; a project-qualified ID syntax would need a registry Meridian doesn't have. The caller's identity is captured before `-C` drops project-scoped environment IDs: restoring those IDs in the target environment would let a child there claim the caller's identity.

## Posted or acknowledged, never resent after a write

The route reports whether the harness acknowledged the request or the message was only posted; neither means the model read it. A missing field from an older launcher reads as posted, and newer builds must keep reading the wire value `acknowledged`. A static per-harness capability would put transport knowledge outside the connection and can't express route-specific outcomes. Codex accepts a steer before its text reaches the rollout (probes found it there within about two seconds), so an immediate session-log check can miss it.

Only a refusal known to come before any write is retried. A timed-out Codex `turn/start` may already have started a turn, so a retry could start a second. An idempotency key wouldn't help, because Claude and Codex can't deduplicate a turn ([connection contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/harness/connections/AGENTS.md)).

## The route stays on the first session

Codex and OpenCode routes stay on the session the primary started with and refuse after detecting a TUI move. Otherwise a message could run in an abandoned session, spend tokens or write files the human never sees, and strand an approval. Following the TUI across `/new` or `/fork` isn't planned: servers don't reliably report TUI focus, and moving the route alone would split it from `--continue`, the idle sensor and the recorded session ID. Detection has gaps, including an OpenCode switch before the human types ([wire evidence](../research/reaching-a-live-tui.md#which-session-is-the-tui-on)).
