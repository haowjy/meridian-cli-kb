# Reaching a Running TUI From Outside: Codex, OpenCode and Pi

Evidence for external input to a running TUI. See [Claude's inbox](claude-code-mods-and-inbox.md) and [Meridian's routes](../decisions/primary-inject.md#unified-delivery).

| | Codex 0.164.0 | OpenCode 2.1.4 | Pi 1.4.0 |
|---|---|---|---|
| Route in | A second app-server client sends `turn/start` or `turn/steer` | `POST /api/session/{id}/prompt` | An extension calls `pi.sendUserMessage`. Pi has no external channel, so that extension has to host its own listener |
| Shows in the TUI | Yes, within about 1 s | Yes, live; the HTTP call itself returned in about 5 ms | Yes, rendered like typed input |
| Session is busy | `turn/steer` with `expectedTurnId` lands in the current turn | `delivery: "steer"` (the default) arrives at the next step boundary; `"queue"` runs as its own turn afterwards | Pass `deliverAs: "steer"` or `"followUp"`; without it the call throws while streaming. Passing it always is safe (see [Pi](#pi)) |
| Model and agent | The thread's settings. A second client's turn uses a TUI `/model` change, though no notification tells that client | The session's committed choice; a TUI `/models` or Tab choice counts only after the TUI submits a prompt | The live model, thinking level, tools and system prompt |
| Approvals | Both clients get the request; either may answer | The TUI shows the prompt; the TUI or `POST .../permission/{reqId}/reply` answers | The same `tool_call` handlers run, so `meridian-permissions` applies |
| What the sender learns | A JSON-RPC result | An HTTP success | Nothing: the call returns `void` |

**What the evidence covers.** Codex and OpenCode were probed live with a plain
websocket client and HTTP calls, not through Meridian's own connection classes
on Linux. Codex's fan-out was also read from its source. Pi was read from the 1.4.0 source
and docs. Later sections add Codex 0.166.0, OpenCode 2.2.5 and Pi 1.5.0, each run live against
a real TUI, and say so where they differ. All of it holds for the versions named and can
change with any release.

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
auto-answer on a primary, or it would beat the human to it. The probe set `approvalsReviewer` to `"user"`; a default launch may behave differently.

`clientId` shows who sent a message: a TUI-typed `userMessage` carries one, and an
injected message from another client has `clientId: null`.

**What a steer does** (0.166.0). The `turn/steer` result returns the current `turnId`, and
that is all an acknowledgement means. The app-server wrote the user line to the rollout
within about 2 s in all six child runs, so a `session log` taken as the send returns can miss it. The model reads the text at its next step, on the same turn: a running command is not
killed, and streaming text is not cut short. A cancelled turn keeps the line, and the next
turn sees it. A model may also ignore it. One child stalled for 171 s and never wrote the
steered text; that did not reproduce, and steered and unsteered runs took about the same
time (43.8 s and 46.1 s). A steer is the same with or without a TUI attached.

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

## Which session is the TUI on?

**No server says.** Codex 0.166.0 has no current-thread method or field, and OpenCode 2.2.5 has
no `/tui/*` route among its 138 `/api/*` routes. A client can only infer a move from
side effects. The move matters because a second client stays on the session it started: a
turn it starts after the TUI has left runs unseen, and an approval that turn raises stays
pending with no prompt on any screen. Claude and Pi don't have this problem, because their
route is inside the TUI process.

The observer sees less than it might seem. Codex sends `thread/started`, `thread/status/changed`,
`thread/name/updated`, `thread/goal/cleared` and `thread/closed` to every client, and `turn/*`
and `item/*` only to a client that started or resumed that thread. OpenCode's
`GET /api/event` carries `session.*` for every session.

### Codex 0.166.0

| What the human does | What the wire shows | Can a client act on it? |
|---|---|---|
| `/new` | `thread/started`: `ephemeral: false`, `parentThreadId: null`, `threadSource: "user"`, `forkedFromId: null`. No `thread/closed` for the old thread | Detectable at once. `thread/resume` fails with `no rollout found` until the TUI's first turn writes the rollout, and an empty new thread is absent from `thread/list` |
| `/fork` | The same, with `forkedFromId` set | The same |
| `/resume` or the sidebar, to a loaded thread | Only `thread/goal/cleared`, which also fires after turns | Weak: nothing else marks it |
| `/resume` to an unloaded thread | `thread/status/changed` to `idle`, then `thread/goal/cleared`, with no `thread/started` | Any client resuming it looks the same |
| `/compact`, `/review` | Same thread id | Not a move |

Things that look like focus and are not:

- `thread/loaded/list` is a set of in-memory ids, helpers included, and the first thread stays
  in it after the TUI leaves.
- `thread/closed` and `notLoaded` mean unloaded. A background thread unloads while the TUI
  is elsewhere.
- A title helper is also a `thread/started`, with `ephemeral: true` and
  `threadSource: "thread_title"`, so "any foreign `thread/started`" is the wrong rule. Guardian
  threads are not ephemeral but carry `parentThreadId` on the wire.

Not seen: a real collab subagent (the schema says `parentThreadId` is set) and a memory helper.
A thread loaded by `thread/resume` announces itself with only status and goal events, so a
subagent reopened that way would read as a move; `thread/read` returns its `parentThreadId` if
that is ever needed.

### OpenCode 2.2.5

| What the human does | What the wire shows |
|---|---|
| New session (`ctrl+x n`) | Nothing, and `GET /api/session` does not list it. The first message then brings `session.created` with no parent |
| Switch session (`ctrl+x l`) | Nothing. The first input there brings `session.inbox.enqueued` and `session.execution.started` naming that session |
| Fork session | `session.forked`, with a `parentID`, and no `session.created` |

Things that look like focus and are not:

- `tui.session.select` is in the event enum and never fired, even on a stream filtered to it.
- `GET /api/session/active` lists running drains. `session.viewed` and
  `POST /api/session/{id}/view` acknowledge an idle transition.
- A fork made through the API moves nothing in the TUI.

A subagent's session arrives as `session.created` with `parentID`. Compaction creates no new
session. The stream is live-only: a reconnect gets `server.connected` and no replay, so a
`session.created` can be lost in a gap. OpenCode V1 was not available to probe.

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
- **Never inject during `session_start`, and don't accept messages there either.**
  Extensions bind before the launch prompt runs, and a run already going makes that prompt
  throw "Agent is already processing"; the TUI shows the error and the prompt is gone. The TUI
  also subscribes later, so it would not show the run. With fast handlers no event-loop turn
  separates `session_start` from the launch prompt (0 iterations over three runs of 43 to 57 ms),
  so a listener can't race it. An extension loaded later that awaits I/O or a dialog in
  `session_start` opens the gap, and a message sent 2.5 s into a 3 s handler did drop the prompt.
  A listener that must not race it binds at the first `agent_start` when a prompt is coming,
  and at `session_start` when none is, or nothing would reach it until the human types.
  `pi.on` inside `session_start` works and returns an unsubscribe function.
- **Load order decides teardown.** CLI `-e` paths load before ambient extensions,
  and `session_shutdown` handlers are awaited in load order, with `managed-bash` waiting on its
  tasks there. A listener loaded after it stays open on a dying session during `/new`,
  `/resume` or `/fork`, and a message sent then was reported `posted` and lost.
- **The factory runs again for every runtime.** Startup, `/new` and `/reload` each called it
  once before `session_start` on 1.5.0, so a listener binds at most once and needs no restart logic.
- **A rejection disappears.** A failed `sendUserMessage` becomes an "Extension error"
  line in the TUI. The extension gets no error, so an outside sender can't be told. A manual `/compact` fails this way, and
  `ctx.isIdle()` does not guard against it. Auto-compaction runs inside the active run, so a
  steer queues (read from source, not run).
- **Always pass `deliverAs: "steer"`.** Choosing from `ctx.isIdle()` is check-then-act: `prompt()`
  awaits `input` handlers and sometimes auth before it re-checks, so a message that saw idle
  throws if a run started meanwhile, and the sender was told `posted`. Pi reads the option only
  while streaming, so an idle message still starts a turn (same logic in 1.1.0 and 1.5.0).
- **`sendMessage` can be matched; `sendUserMessage` cannot.** `sendMessage` carries
  `customType` and `details`, which survive into `message_start` and `message_end`,
  but the TUI shows a custom block rather than a user turn. `sendUserMessage` is
  indistinguishable from typed input.
- **Leave `expandPromptTemplates` false.** True would run extension commands from
  outside text.
- **Closing a listener unlinks by name.** A second listener on the path loses its socket when
  the first closes.

From source, not run: a steered message in a run the human aborts goes back to the editor.
