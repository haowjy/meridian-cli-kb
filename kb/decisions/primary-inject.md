# Primary message delivery

Every primary kind serves the same control socket used by spawned sessions. The
[message route contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/harness/.context/CONTEXT.md#primary-message-route),
[message framing contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/cli/.context/CONTEXT.md), and
[user-facing harness details](https://github.com/haowjy/meridian-cli/blob/main/docs/harness-integration.md)
describe current behavior. The [TUI research](../research/reaching-a-live-tui.md)
records the harness evidence; [#916](https://github.com/haowjy/meridian-cli/issues/916)
tracks remaining edges.

## Claude primary inbox {#d-claude-primary-inbox}

Claude's inbox was chosen over a TUI mod because it can deliver between tool
calls, needs no additional listener and does not depend on a mod loading silently.
A mod cannot replace the child stdin stream: plain `claude -p` exits after going
idle even if a mod listener remains, and `$.prompt.submit` waits for the turn to
end. The starting direction was to use a mod for both children and primaries; the
owner has not confirmed the departure from that direction or the approval hold
on bypass-mode primaries. See the [probes and inbound controls](../research/claude-code-mods-and-inbox.md).

Meridian does not send Claude's messaging token, forge a bypass-mode envelope,
or override `crossSessionInbound`: those would claim authority or override a
choice the user has not given Meridian. The inbox gives no receipt that
separates held, refused and delivered messages. A user can choose
`crossSessionInbound: "accept"`; Meridian does not choose it for them. The inbox
schema comes from the Claude binary and can change without notice.

## Message frame {#d-spawn-message-frame}

The owner asked for a reply ID: “there is no id for the response... so we should
probably add like 'this was sent from a meridian session. use `meridian spawn
message c9297 etc...'”. The framing and route check are current implementation
choices. Sender identity travels in text because every route carries it, unlike
Claude's peer envelope. The reply uses the sender's current chat, rather than
its spawn ID or the chat that launched it: continuation changes the spawn ID,
and a session can move to another chat. A reply command is offered only when a
socket file exists; that is a reachability indication, not a successful-send
guarantee.

## Message address {#d-spawn-message-address}

The owner asked for “an easier way to do cross/repo communication” and for
`spawn message` to be more accessible. Chat and spawn IDs are project-scoped,
so cross-project addresses include the
project root. A project-qualified ID syntax would require a registry Meridian
does not have. The caller identity is captured before `-C` drops project-scoped
environment IDs; restoring those IDs in the target environment would let a child
there claim the caller's identity. See [chat ownership](chat-spawn-ownership.md#d-chat-names-own-session).

## Delivery acknowledgement {#d-delivery-kind}

The route reports whether a harness acknowledged the request or the message was
merely posted. Neither means the model read it. A missing field from an older
launcher is treated as posted, and the wire value `acknowledged` must remain
readable by newer builds. A static harness capability would put transport
knowledge outside the connection and cannot represent route-specific outcomes.

Codex accepts a steer before its text appears in the rollout. Probes found it in
the rollout within about two seconds and at the model's next step, including
after cancellation, but an immediate session log can miss it. OpenCode V1's
acknowledgement was assessed from code, without a running V1 server. See
[delivery research](../research/reaching-a-live-tui.md#codex).

## No retry after a possible write {#d-no-retry-after-write}

Only a refusal known to precede a write is retried. A timed-out Codex
`turn/start` may already have started a turn, so retrying can start another.
A fake connection reproduced the double send; the duplicate turn was inferred,
not forced against Codex. An idempotency key was rejected because Claude and
Codex cannot deduplicate a turn. See the
[connection contract](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/harness/connections/AGENTS.md).

## Unified delivery {#d-unified-delivery}

The launcher serves a common socket instead of adding a CLI branch and separate
reachability check for each primary harness. The older split had already allowed
a reply command for a Claude primary without a usable inbox. The socket is bound
when a route exists rather than predicting support from launch flags, version,
or an outstanding trust prompt. It can still be stale or refuse a send.

Codex and OpenCode's observer became load-bearing: it must survive idle silence
because it is also the message route. Meridian does not answer approvals raised
by injected turns; those reach the human's TUI. Pi serves the socket in an
extension because it has no external delivery channel. A launcher-to-extension
socket would add a second transport; a file drop would not report failure.
Maintaining Pi's wire handling in TypeScript is the tradeoff.

The owner accepted that older primaries need relaunching, that OpenCode counts
injected turns as activity, that a busy Codex primary is steered, and that
Claude's inbox adds its peer-safety paragraph. A no-route primary can therefore
wait through discovery before failing.

## Session moves {#d-primary-session-move}

Codex and OpenCode routes stay on the first session and refuse after detecting a
TUI move. Otherwise a message can run in an abandoned session, spend tokens or
write files while the human sees nothing, and leave an approval stranded. The
owner accepted refusal and does not expect Meridian to work well across `/new`
or `/fork`; following the TUI is not planned. A partial follow was rejected:
servers do not reliably report TUI focus, and moving the route alone would split
it from `--continue`, the idle sensor and the recorded session ID. Detection has
gaps, including an OpenCode switch before the human types; see
[#916](https://github.com/haowjy/meridian-cli/issues/916) and the
[wire evidence](../research/reaching-a-live-tui.md#which-session-is-the-tui-on).
