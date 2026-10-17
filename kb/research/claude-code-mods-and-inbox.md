# Claude Code: Mods Under `-p`, and the Cross-Session Inbox

What Claude Code actually does when something outside a running session tries to
give it input: through a Claude Code mod (`$.prompt.submit`, a listener process,
hooks), or through Claude's own per-session inbox socket. Checked 2026-10-17
against Claude Code 2.1.302 with Haiku, on Linux, by live probes in a dedicated
tmux server. Provenance: `work:claude-mod-inject` (`probe-mod-inject.md`
`spawn:p8410`, `probe-tui-mod.md` `spawn:p8415`, `probe-inbox-socket.md`
`spawn:p8419`, research `spawn:p8416`).

**The short version.** A mod cannot keep a print-mode (`-p`) process alive for
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
| `session.start` | Fires once per process, before the first prompt. Not on `/clear` or `/compact`. A resume is a new process, so it fires again. A listener started there with `$.process.spawn` dies with the TUI. Contrast the classic `SessionStart` command hook, which does fire on `/clear`. |
| A mod that fails to load | Silent: a syntax error still gives exit 0, empty stderr and a normal answer. Detect loading with a ready marker written from `session.start`. |
| Loading | `--plugin-dir` and `CLAUDE_CODE_PLUGIN_DIRS` both load a mod. |
| Mod tools (`$.tool.register`) | The model calls `mcp__<plugin>__<tool>` with no permission prompt, in bypass and default modes. |
| Ending the session from a mod | No API. `$.turn.abort` exists; nothing exits the process, so an outside watcher has to kill it. |
| `$.ui.status` | Shows beside a user's own status line; both stay visible. |
| `$.session.send` from a second process to a TUI session | Delivered (`isDelivered:true`), and the TUI answered. |

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

This is the SDK stream-json user message. The docs describe only the auth line
and say scripts may post; the message schema was read from the 2.1.302 binary.
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
through.

- **A receiver in default or auto mode delivers** a bare post.
- **A receiver that bypasses permission prompts** (`--dangerously-skip-permissions`)
  **holds a sender that did not attest its permission mode**, and shows a screen
  offering "Deny" or "Deliver this message to Claude". This is a hold-review
  screen, not a tool permission prompt.
- **Attestation is a content envelope**, not a JSON field. A post whose `content`
  is `<cross-session-message from-mode="bypass">`, a newline, the body, a newline,
  and `</cross-session-message>` delivers to a bypass receiver unattended. The
  newlines are required. An outer `"from_mode":"bypass"` key attests nothing.
  Other envelope attributes: `from` (a `uds:` reply address), `from-session`,
  `from-name`, `from-plugin`, `hop-chain`.
- **`crossSessionInbound: "accept"`** delivers unattested posts to a bypass
  receiver.
- **The token claims own-child status.** Claude treats a sender presenting the
  messaging token as a process it started; on macOS that skips the inbound
  controls (research `spawn:p8416`; not probed on macOS).

**Trap: `--settings` outranks user settings.** Passing `crossSessionInbound`
through a launch's `--settings` would override a value the user set in their own
settings, including `refuse`.

---

## Not covered

- macOS behaviour of unauthenticated posts.
- Hot reload of a mod.
- A settings-file command `Stop` hook returning `block` under `-p`.
- Claude Code versions other than 2.1.302. The inbox needs a version that
  publishes the socket; Meridian's error message names 2.1.224 as the minimum.
  Bare mode has no socket.
