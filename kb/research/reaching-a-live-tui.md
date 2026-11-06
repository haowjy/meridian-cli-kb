# Reaching a Running TUI From Outside: Codex, OpenCode and Pi

What each harness accepts from a second process while a human's TUI is attached to
the session. Read this before building a message route to a primary. Claude's route
is on its own page, [Claude Code: mods and the inbox](claude-code-mods-and-inbox.md).
No Meridian build uses these routes for a primary yet; the plan that would is
[D-unified-delivery](../decisions/primary-inject.md#d-unified-delivery).

| | Codex 0.164.0 | OpenCode 2.1.4 | Pi 1.4.0 |
|---|---|---|---|
| Route in | A second app-server client sends `turn/start` or `turn/steer` | `POST /api/session/{id}/prompt` | An extension calls `pi.sendUserMessage`; nothing reaches the process from outside |
| Shows in the TUI | Yes, within about 1 s | Yes, the call returns in about 5 ms | Yes, rendered like typed input |
| Session is busy | `turn/steer` with `expectedTurnId` lands in the current turn | `delivery: "steer"` (the default) arrives at the next step boundary; `"queue"` runs as its own turn afterwards | Pass `deliverAs: "steer"` or `"followUp"`; without it the call throws while streaming |
| Model and agent | The thread's settings, including a TUI `/model` change, with no notification to other clients | The session's committed choice; a TUI `/models` or Tab choice counts only after the TUI submits a prompt | The live model, thinking level, tools and system prompt |
| Approvals | Both clients get the request; either may answer | The TUI shows the prompt; the TUI or `POST .../permission/{reqId}/reply` answers | The same `tool_call` handlers run, so `meridian-permissions` applies |
| What the sender learns | A JSON-RPC result | An HTTP success | Nothing: the call returns `void` |

**What the evidence covers.** Codex and OpenCode were probed live with a plain
websocket client and HTTP calls, not through Meridian's own connection classes
(`spawn:p9741`, Linux). Codex's fan-out was also read from its source
(`spawn:p9723`). Pi was read from the 1.4.0 source and docs and never run
(`spawn:p9736`). All three hold for the versions named and can change with any
release.

## Codex

The app-server serves many clients on one thread, and `thread/resume` only
subscribes. A client that started the thread keeps its websocket open after the TUI
attaches with `codex resume <id> --remote`, and it keeps receiving notifications for
turns the TUI starts. A new client can also join a loaded thread with `initialize`
and `thread/resume`, then send, steer and answer approvals; the TUI keeps working.
So a holder of the observer connection needs no reconnect to send.

Approvals are a race. `item/commandExecution/requestApproval` reaches every
subscribed client, the TUI shows its own prompt, and the first answer wins; the other
client gets `serverRequest/resolved` and its prompt clears. Meridian must never
auto-answer on a primary, or it would beat the human to it. The probe set
`approvalsReviewer` to `"user"` to get real prompts, because this user's Codex config
defaults to `auto_review`, so a default launch may behave differently.

`clientId` tells turns apart on the wire: a TUI-typed `userMessage` carries one and
another client's injected message has `clientId: null`.

## OpenCode

The session prompt endpoint is the only injection route. No `/tui/*` or `/api/tui/*`
route exists, and that bears on how you test: a GET on an unknown path returns 200
`text/html` from the single-page app, so a 200 on a guessed path proves nothing.
Check the content type. An injected prompt leaves a half-typed TUI draft alone.

Pending messages appear in `GET .../inbox`. A steer can replace the reply the session
was about to give: in the probe the original answer never came.

The server prints a password at startup, and the HTTP API needs it as basic auth
`opencode:<password>`. Meridian never writes that password to disk, so only the
launcher that started the server can post.

## Pi

Pi has no external channel into a running TUI: no socket, no `pi send` or `pi attach`,
and no watch on the session file. RPC mode is a separate headless mode, not a sidecar.
A route therefore needs an extension in the Pi process to host a listener or call the
API. Pi's own example of an external trigger is a file watcher,
`examples/extensions/file-trigger.ts`.

- **Lifetime.** Start the listener in `session_start` and close it in an idempotent
  `session_shutdown`. The factory also runs when no session starts. Guard on
  `ctx.mode === "tui"`, because a listener that outlives shutdown hangs print mode.
- **Crashes leave the socket.** `emergencyTerminalExit` and `uncaughtCrash` emit no
  `session_shutdown`, so the listener has to unlink on start and handle `EADDRINUSE`.
- **Never inject during `session_start`.** Extensions bind before the launch prompt
  runs, so an injection there makes that prompt throw "Agent is already processing"
  and drops it. The TUI also subscribes later, so it would not show the run.
- **A rejection disappears.** A failed `sendUserMessage` becomes an "Extension error"
  line in the TUI and never returns to the caller. Compaction refuses a prompt this
  way. Check `ctx.isIdle()` first to choose `deliverAs`.
- **`sendMessage` can be matched; `sendUserMessage` cannot.** `sendMessage` carries
  `customType` and `details`, which survive into `message_start` and `message_end`,
  but the TUI shows a custom block rather than a user turn. `sendUserMessage` is
  indistinguishable from typed input.
- **Leave `expandPromptTemplates` false.** True would run extension commands from
  outside text.

A Pi extension does not know Meridian's runtime root, and the control socket's path
is derived from it. Only `MERIDIAN_SPAWN_ID` and `_MERIDIAN_PI_*` values reach the
child today, so an extension serving that socket would need a new variable.
