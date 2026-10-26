# Claude Code: Mods Under `-p`, and the Cross-Session Inbox

How a process outside a running Claude Code session can give that session new
input, and what each route does. Two routes were tested: a **mod** (a Claude Code
plugin that runs JavaScript inside the session, with an API under `$`, such as
`$.prompt.submit` to submit a prompt; [mods API](https://code.claude.com/docs/en/plugins/mods/api)),
and Claude's own per-session **inbox socket**, a Unix socket other sessions post
messages to ([cross-session messaging](https://code.claude.com/docs/en/cross-session-messaging#the-sessions-inbox-socket)).
Both were tried in print mode (`claude -p`, one prompt, then exit) and in the
interactive TUI. Checked 2026-10-17
against Claude Code 2.1.302 with Haiku, on Linux, by live probes in a dedicated
tmux server. Provenance: `work:claude-mod-inject` (`probe-mod-inject.md`
`spawn:p8410`, `probe-tui-mod.md` `spawn:p8415`, `probe-inbox-socket.md`
`spawn:p8419`, research `spawn:p8416`).
Later-version findings in the table are explicitly versioned and cover only
`$.ui.status` behavior probed for the rejected status-row design in
[#706](https://github.com/haowjy/meridian-cli/pull/706). They describe the host
API, not a Meridian feature. [Mod hooks for idle delivery](#mod-hooks-for-idle-delivery-21307)
was checked on 2.1.307 for the Claude spawn watcher (`work:claude-spawn-watch`).
The `crossSessionInbound` values, hold causes and settings precedence under
[Inbound controls](#inbound-controls) were read from the 2.1.310 bundle and
matched against the owner's own held messages (`spawn:p9612`, PR #777).
The other observations remain scoped to 2.1.302.

**The short version, for 2.1.302.** A mod cannot keep a print-mode (`-p`) process alive for
input, and a mod's `$.prompt.submit` waits until the current turn ends. The inbox
socket reaches a live interactive session with no mod at all and delivers between
tool calls in the middle of a turn, but it answers nothing back, and its inbound
security rules hold messages for a session that bypasses permission prompts.
Meridian's choices built on these facts are in
[D-claude-primary-inbox](../decisions/primary-inject.md#d-claude-primary-inbox)
and [D-claude-native-hold](../decisions/completion-drain.md#d-claude-native-hold).

---

## Mods

| Behaviour | Observed |
|---|---|
| Plain `claude -p` once its turn is idle | Exits after about 2.5 s, exit 0. A live `$.clock.every` timer or an un-awaited `$.process.spawn` child does not keep it open; a message written at T+15 s found the process gone. |
| Claude's own background work under `-p` | Kept alive correctly. `-p` waits for a background shell (`run_in_background`) and a background subagent, runs the follow-up turn, emits the final `result`, then exits 0. |
| Holding `-p` open for new input | Nothing found. A mod `classic.Stop` hook can delay exit while it waits, but returning `{decision:"block", reason}` starts no new turn. `turn.complete` only delays exit. `$.session.send` to the session's own id returns `isDelivered:false`. A settings-file command `Stop` hook (`--settings`) returning `block`, the documented continue mechanism, was not tested. |
| `$.prompt.submit` during a turn | Accepted at once, but the submitted turn starts only after the current turn, including a long tool call, finishes. |
| `$.prompt.submit` in a TUI while idle | First assistant text about 1.7 s after the send. The user's unsent composer text is kept. The transcript row is `type:"user"` with `origin.kind:"plugin"` and wrapped "…plugin sent a message" text. |
| Mod-started turns in `-p` stdout | No standalone user frame for the submitted text. The `result` carries `origin.kind:"plugin"`; the last `type:"result"` frame is the reliable final report. |
| `session.start` | Fires once per process, before the first prompt. Not on `/clear` or `/compact`. A resume is a new process, so it fires again. A listener started there with `$.process.spawn` dies with the TUI. Contrast the classic `SessionStart` command hook, which does fire on `/clear`. Also fires for every later fresh load of one plugin (see [Mod hooks for idle delivery](#mod-hooks-for-idle-delivery-21307)). |
| A mod that fails to load | Silent: a syntax error still gives exit 0, empty stderr and a normal answer. Detect loading with a ready marker written from `session.start`. |
| Loading | `--plugin-dir` and `CLAUDE_CODE_PLUGIN_DIRS` both load a mod. |
| Mod tools (`$.tool.register`) | The model calls `mcp__<plugin>__<tool>` with no permission prompt, in bypass and default modes. |
| Ending the session from a mod | No API. `$.turn.abort` exists; nothing exits the process, so an outside watcher has to kill it. |
| `$.ui.status` | Creates a persistent plugin row beside a user's own status line; both stay visible, and the host supplies the plugin-name prefix. On Claude Code 2.1.303, `$.ui.status(undefined)` removed the row while `$.ui.status("")` left a named blank row; 2.1.306 confirmed removal with `undefined`. These checks were part of rejected PR #706, not current Meridian behavior. |
| `$.session.send` from a second process to a TUI session | Delivered (`isDelivered:true`), and the TUI answered. |

---

## Mod hooks for idle delivery (2.1.307)

Facts a mod that starts turns at idle depends on, from building and probing
Meridian's `meridian-spawn-watch` mod on Claude Code 2.1.307 (live Haiku TUI
sessions, the host's `claude-code.d.ts`, and `claude plugin test`). How
Meridian uses them: [Child result delivery](../concepts/child-result-delivery.md).
"Main loop" below means the session's own conversation, as opposed to a
subagent's; its turn events carry no `agentId`.

| Behaviour | Observed |
|---|---|
| `$.prompt.submit` resolution | Resolves when the turn starts, when the prompt is queued behind a running turn, or with `{ drop }`. A resolved submit therefore does not prove a turn ran: a queued prompt can still be lost when the queue is cleared (Esc, crash). To act once the model has the text, put a nonce in it and wait for the main-loop `turn.start` whose text contains the nonce. |
| Submitted text in `turn.start` | The host wraps it ("The meridian-spawn-watch plugin sent a message:" … an explanatory footer); `turn.start` carries the whole wrapped text, so a substring match on the nonce works. |
| `session.start` per load | Fires for each loaded plugin before the first prompt, and is awaited; then once per fresh load of one plugin (enable, worker respawn, reload), running that plugin's hooks alone. Every copy of a module receives it before its other hooks, so a "loaded without `session.start`" fallback guards a state the host never produces. |
| `/clear` | Fires `session.end` and no `session.start`. |
| Typing in the prompt box | Typing, pasting and deletion (including a kill such as C-u) arrive as `prompt.edit`. `prompt.change` carries only engine writes: history recall, a clear, a plugin's fill. A "user touched the box recently" check needs both hooks. With only `prompt.change`, clearing a draft with C-u looked untouched and a delivery landed at once. |
| Subagent turns | `turn.start` and `turn.complete` fire for subagents, teammates and compaction forks too, with `agentId` set. A main-loop hook must filter on `agentId === undefined`, or background agents keep restarting it. |
| `turn.complete` reasons | `answer` for a normal end and `aborted` after Esc; errors (API, rate or context limits) and refusals carry other reasons. Gate "the turn ended normally" on `answer` alone. |
| `$.process.spawn` cleanup | Leaving the read loop kills the child, and `stream.return()` works while a read is pending (SIGTERM within the same second on a blocked child). Racing `stream.next()` against an `AbortController` gives a cancellable child. |
| Hot reload (edit under `--plugin-dir`) | The engine kills the old copy's running child process, prints its own "reloaded" notice, and the new copy starts with clean state and gets `session.start`. |
| Sharing code between plugins | `claude plugin validate` and `test` refuse an import from outside the plugin's folder ("outside the plugin's folder"), and a symlink to a sibling directory is not followed. Each plugin holds all of its own source. |
| Test kit | `mock.clock` mocks `$.clock.now()`, not `Date.now()`. `clock.advance` does not settle while a `process.spawn` stream is pending. `prompt.edit`/`prompt.change` have no typed call on `$`, but calling them through a cast raises them at runtime. One `on('x')` per event. |
| Module shape | Helpers that take `$` must be top-level function declarations, or the module validator rejects them. The hooks environment has `AbortController` but no `setTimeout`. |

---

## The inbox socket

Every session listens on a Unix socket for messages from other sessions. Claude
exports its path to hooks before `SessionStart` as `CLAUDE_CODE_MESSAGING_SOCKET`
(documented), with a token in `CLAUDE_CODE_MESSAGING_TOKEN`. The default path is
`cc-socks/<claude-pid>.sock` under `$XDG_RUNTIME_DIR`, or under `/tmp` without
it. A hidden, undocumented `--messaging-socket-path` flag overrides it.

**Lifetime.** `/clear` kept the socket path and token (tested with a path set by
the flag) while the session id changed. Exiting and resuming the same session
gives a new process, so a new default path and a new token under the same
session id. Anything that records the path has to tie it to the process that
published it.

**Wire format.** Newline-delimited JSON. An optional first line
`{"type":"auth","token":"…"}`, then one line per message:

```json
{"type":"user","message":{"role":"user","content":"<non-empty string>"}}
```

This is the SDK stream-json user message. The cross-session messaging docs
describe only the auth line and say scripts may post; the message schema was read from the 2.1.302 binary.
On Linux a post with no auth line, from a process that is not Claude's child, was
accepted and delivered.

**No reply.** The socket writes nothing back on the posting connection: a client
that half-closes and reads gets EOF. Delivered, held and refused look identical.
Claude does send status receipts
(`{"type":"control","action":"peer_message_status","status":"held|denied|expired|delivered|refused|dropped",…}`),
but only to a reply address a real sender names in the message envelope (below).
That receipt format is also binary-only.

**Delivery timing.**
- Idle: the post starts a turn; the reply began within about 1 s.
- Mid-turn: the message is absorbed *between tool calls in the same turn*. In a
  turn of three 6-second shell calls, a post sent after call 1 was read after
  call 2 and before call 3. The transcript records
  `queue-operation … reason:"absorbed_mid_turn"`, and the model sees it as a
  system reminder rather than a new user row.
- Five concurrent posts were all delivered, but not in send order, and the model
  answered four of them in one response.

**Framing.** The TUI shows "Another Claude session sent a message". The
transcript row is `type:"user"`, `isMeta:true`, `turnOrigin:"peer"`,
`origin.kind:"peer"`, and for a bare post `origin.from:"unknown"` with the
sender's verified OS pid and start time. The model is told the message came from
another Claude session, not from its user.

### Inbound controls

The receiving session's `crossSessionInbound` setting governs whether posts get
through: `accept` delivers straight to the session's queue, `hold` parks every
peer message for review, `refuse` turns peer messages away.

- **Unset, Claude runs a permission-mode parity check.** A receiver in default
  mode delivers a bare post. A receiver that bypasses permission prompts holds
  the post when the sender runs in a different permission class (cause
  `mode-mismatch`) or attested no mode at all (cause `no-mode-asserted`).
  Bypassing includes `--dangerously-skip-permissions` and
  `permissions.defaultMode: "auto"`: a session in auto mode holds every
  unattested post (Claude Code 2.1.310, observed on the owner's sessions).
- **The hold shows a review screen** offering "Deny" or "Deliver this message to
  Claude", not a tool permission prompt. Approving it reports "Released 1 held
  cross-session message to Claude's queue (you approved it)". A held message
  releases only on approval or expires; a headless host has no approval surface,
  so its held messages expire.
- **Repository and managed settings can tighten the value, not loosen it.** A hold
  that a standing setting causes carries `explicit-setting`, `repo-setting` or
  `managed-setting` instead of a parity cause.
- **Attestation is a content envelope**, not a JSON field. A post whose `content`
  is `<cross-session-message from-mode="bypass">`, a newline, the body, a newline,
  and `</cross-session-message>` delivers to a bypass receiver unattended. The
  newlines are required. An outer `"from_mode":"bypass"` key attests nothing.
  Other envelope attributes: `from` (a `uds:` reply address), `from-session`,
  `from-name`, `from-plugin`, `hop-chain`.
- **`crossSessionInbound: "accept"`** delivers unattested posts to a bypass
  receiver. It applies to every peer message the session receives, so a
  bypassing session then acts on any other session's message without review.
- **The token claims own-child status.** Claude treats a sender presenting the
  messaging token as a process it started; on macOS that skips the inbound
  controls (research `spawn:p8416`; not probed on macOS).

**Trap: `--settings` outranks user settings.** Passing `crossSessionInbound`
through a launch's `--settings` would override a value the user set in their own
settings, including `refuse`.

---

## Not covered

- macOS behaviour of unauthenticated posts.
- A settings-file command `Stop` hook returning `block` under `-p`.
- Claude Code versions other than 2.1.302, apart from the separately noted
  rejected-surface `$.ui.status` checks on 2.1.303 and 2.1.306, the 2.1.307
  idle-delivery hooks, and the 2.1.310 inbound-gate reading. The inbox needs a version that
  publishes the socket; Meridian's error message names 2.1.224 as the minimum.
  Bare mode has no socket.
