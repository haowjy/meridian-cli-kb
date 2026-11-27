# Messaging a live session

`meridian message ADDRESS "text"` reaches every live session, primaries included, through the same control socket spawned sessions serve. The address says where a message goes: a spawn or chat ID for a session on this machine, `<machine>/<id>` for one on a [saved machine](../machines.md), `@person` for a person, `inbox` for the machine inbox. One verb means an agent doesn't need to know which program owns the receiver before it can say anything. The command surface, error kinds and JSON are in meridian-cli `docs/commands.md`. Current behavior: the source checkout's [message route contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/harness/.context/CONTEXT.md#message-delivery), [framing contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/cli/.context/CONTEXT.md) and `docs/harness-integration.md`. Harness evidence: [reaching a live TUI](../research/reaching-a-live-tui.md). Remaining edges: [#916](https://github.com/haowjy/meridian-cli/issues/916).

## One socket for every primary

The launcher serves a common socket instead of a CLI branch and reachability check per harness. The socket is bound once a route actually exists, rather than predicting support from launch flags, version or an outstanding trust prompt; it can still be stale or refuse. A sender may wait for the route to appear before failing. Advertising a reply route with no usable session inbox would offer a command that can't deliver. Older primaries need relaunching to expose the route.

- **Codex and OpenCode:** the observer that watches the backend is also the message route, so it must survive idle silence. A busy Codex primary is steered; OpenCode counts injected turns as activity. Meridian doesn't answer approvals raised by injected turns; they reach the human's TUI.
- **Pi:** an extension serves the socket, because Pi has no external delivery channel. A launcher-to-extension socket would add a second transport, and a file drop couldn't report failure; the cost is maintaining the wire handling in TypeScript.
- **Claude:** the launcher's socket writes into Claude's cross-session inbox, below.

## Claude primaries use the session inbox

Claude's cross-session inbox delivers between tool calls, needs no extra listener, and doesn't depend on a mod loading silently. A mod also can't replace the child's stdin stream: plain `claude -p` exits after going idle even if a mod listener remains, and `$.prompt.submit` waits for the turn to end ([probes](../research/claude-code-mods-and-inbox.md)). The inbox schema comes from the Claude binary and can change without notice, and the inbox gives no receipt that separates held, refused and delivered. Claude adds its peer-safety paragraph to injected messages.

Meridian doesn't send Claude's messaging token, forge a bypass-mode envelope, or override `crossSessionInbound`: each would claim authority the user hasn't given Meridian. A user can set `crossSessionInbound: "accept"` themselves.

> [!FLAG] **Needs human review**: whether to replace Claude's session inbox with a mod, and whether bypass-mode primaries should hold injected messages for approval, are undecided. Flagged 2026-11-07.

## The frame carries a reply address

Sender identity travels in the message text, because every route carries text, unlike Claude's peer envelope. The reply goes to the sender's current chat, not its spawn ID or launching chat: a session can move to another chat (`/clear`, native `/resume`), so its spawn ID or launching chat may no longer reach the conversation the sender is in. A reply command is offered only when a socket file exists, which shows reachability, not that a send will succeed.

A claimed chat ID reaches its session from any project, so the reply line is a bare `meridian message cN` ([machine-wide IDs](../launch/session-references.md#ids-are-machine-wide)); it adds `-C` only for an ID issued before claims, and a machine prefix when the sender is on another machine, written with the name the receiver saved for it. The caller's identity is captured before `-C` drops project-scoped environment IDs: restoring those IDs in the target environment would let a child there claim the caller's identity.

## The command leaves the sandbox by its first words

A control socket, `ssh` and the people program are all out of a harness sandbox's reach, so `meridian message` runs outside it, as `meridian spawn` does, and only for agents whose profile may spawn: no session gains or loses the ability to send ([sandbox projection](../permissions/sandbox-projection.md)). Claude and Codex match the first words of a command, so every reply line, help text and skill puts global options after the address (`meridian message cN -C <path> "…"`); written `meridian -C <path> message …`, the command stays sandboxed and can't reach the socket. A sandbox is fixed when the session starts, so a session started before the grant existed has to be restarted to send with the new command.

## A message to a cold session is held

A cold session is idle past its prompt-cache deadline, so the turn a message starts re-reads the whole conversation at full price. `meridian message` to one sends nothing and fails with kind `cold`, giving how long it has been idle, its model, its cost so far, and the two ways on: send again with `--wake`, or start a fresh session with `--from`. The owner chose to warn instead of waking, so the sender weighs the cache miss first; often a fresh session started `--from` the cold one costs less. The check runs before any byte is written, so `cold` always means nothing arrived. A session that reports no idle state is delivered to, as before: the hold is best effort, not a gate. The warning gives no cost estimate, because no harness exposes the current context size.

`--wake` belongs to a session address. Meridian's own `agy` never-delivered notice passes it: that notice is a failure report the sender is owed, and holding it would drop it. [meridian-comms](../ecosystem.md#meridian-comms) passes it when a person chose the session.

A delivery that starts a turn ends the session's idle stretch, and that turn opens a new one with its own push and warning, because the session did real work. Only a spawned child's control server ends the stretch on delivery, and only for `posted` or `acknowledged`. A primary's stretch is left to its idle sensor, which sees a turn when it really starts: a Claude primary that bypasses permission prompts holds an inbound message for approval, so `posted` doesn't mean a turn began, and ending the stretch then would silently drop the human's idle push ([idle notifications](idle-notifications.md)).

## People go to a program the user names

Meridian knows nothing about Slack: a second chat surface inside Meridian would duplicate what [meridian-comms](../ecosystem.md#meridian-comms) does. An `@` address goes to the program in `[message] people`, which Meridian runs with no shell and reads one line of JSON back from. Meridian owns that contract, so it has one printer and one list of error kinds, and any program can implement it; passing a tool's own terminal text through would leave Meridian unable to tell a post that went out from one that didn't. The program is not found on `PATH` by a built-in name, which would send to Slack on one machine and fail on another with nothing written down to say why.

`[message]` is read only from the user's `~/.meridian/config.toml`. A project's `meridian.toml`, a local override or a `--config` file is repository content a sandboxed session can write, and the message runs outside the sandbox: `people = ["sh", "-c", …]` there would run a program of the repository's choosing. Who may post stays the program's decision; Meridian passes the starting environment through so the program can tell a child from a primary, and an `@` address never crosses machines, because a command arriving from another machine carries no session variables and would read as a person at a terminal.

## Only one error kind means it may have arrived

Every outcome has a kind, and only `maybe_delivered` means the message may have arrived; with any other, nothing was sent and sending again is safe. Once the first byte is written, an outcome without a parsed answer is `maybe_delivered`, and so is a reply with a kind this build doesn't know. Scripts such as meridian-comms poll and resend, and a resend after a partial write could deliver twice, which a turn can't undo.

## Posted or acknowledged, never resent after a write

The route reports whether the harness acknowledged the request or the message was only posted; neither means the model read it. A missing field from an older launcher reads as posted, and newer builds must keep reading the wire value `acknowledged`. A static per-harness capability would put transport knowledge outside the connection and can't express route-specific outcomes. Codex accepts a steer before its text reaches the rollout (probes found it there within about two seconds), so an immediate session-log check can miss it.

Only a refusal known to come before any write is retried. A timed-out Codex `turn/start` may already have started a turn, so a retry could start a second. An idempotency key wouldn't help, because Claude and Codex can't deduplicate a turn ([connection contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/harness/connections/AGENTS.md)).

## The route stays on the first session

Codex and OpenCode routes stay on the session the primary started with and refuse after detecting a TUI move. Otherwise a message could run in an abandoned session, spend tokens or write files the human never sees, and strand an approval. Following the TUI across `/new` or `/fork` isn't planned: servers don't reliably report TUI focus, and moving the route alone would split it from `--continue`, the idle sensor and the recorded session ID. Detection has gaps, including an OpenCode switch before the human types ([wire evidence](../research/reaching-a-live-tui.md#which-session-is-the-tui-on)).
