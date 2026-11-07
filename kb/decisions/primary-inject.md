# Decisions: Primary Inject

How `meridian spawn message` reaches a live session, whether a spawned child or a
**primary** (a session a human started in a harness TUI). Every session takes messages
on one control socket ([D-unified-delivery](#d-unified-delivery)). A Codex or OpenCode
primary's route stops once its TUI leaves the first session
([D-primary-session-move](#d-primary-session-move)). Every message names its sender
([D-spawn-message-frame](#d-spawn-message-frame)), one address form reaches a session in
this project or another ([D-spawn-message-address](#d-spawn-message-address)), and the
sender learns how far its message got ([D-delivery-kind](#d-delivery-kind),
[D-no-retry-after-write](#d-no-retry-after-write)).
[D-claude-primary-inbox](#d-claude-primary-inbox) records why Claude's inbox is the way
into a Claude primary. The code contract is in meridian-cli
`src/meridian/lib/harness/.context/CONTEXT.md` ("Primary message route"); what each harness
accepts from outside is in [Reaching a running TUI](../research/reaching-a-live-tui.md) and
[Claude Code: mods and the inbox](../research/claude-code-mods-and-inbox.md).
The command was `spawn inject` until PR #777. "Inject" survives only as a word for a message
sent into a running session.

---

## D-claude-primary-inbox: a Claude primary is injected through Claude Code's own inbox socket, not a mod (2026-10) {#d-claude-primary-inbox}

**Status:** decided by the tech lead on probe evidence (`work:claude-mod-inject`
D2, D6–D9). Issue #665; merged in PR #668 (v0.10.22). Before it, the command
against any primary polled 30 s for a control socket that never appeared, then
failed with `spawn not running: pN has no control endpoint`.

Decisions 1 to 4 stand. [D-unified-delivery](#d-unified-delivery) replaced decisions 5
and 6: the launcher now posts to the inbox, the `primary_inbox` port and the CLI's branch
are gone, a primary takes the same 30 s discovery wait as a child, and the PID-reuse guard
went because a launcher serving the socket owns its child.

> [!FLAG] **Needs human review.** The owner's direction was a Claude Code mod
> that owns inject, so that Claude children could drop the stdin input stream
> and Claude could handle its own idle and background work as a black box
> (`chat:c8112`: "I mean move inject into a mod etc. and then we don't need the
> input stream anymore"). The probes below showed a mod cannot replace a child's
> stdin. The tech lead reported that result to the owner, who did not rule on it.
> For primaries the tech lead then chose Claude's inbox socket over a mod. The
> owner has not confirmed either outcome, or the approval dialog that bypass-mode
> primaries now show for each inject (decision 3).

### Problem

A Claude primary runs as a TUI behind Meridian's PTY relay. Before this decision
the command against one waited 30 s and then reported a live primary as "not
running" (`spawn:p8415`: 30.8 s). An agent or a user had no way to hand a running
Claude TUI a message from outside it.

The starting direction was one Meridian mod for both kinds of Claude session.
Children would run as plain `claude -p` with no `--input-format stream-json`,
the mod would listen for injected messages and submit them with
`$.prompt.submit`, and TUI primaries would load the same mod. Three probes on
Claude Code 2.1.302 settled what is possible; their facts are in the
[research page](../research/claude-code-mods-and-inbox.md).

### Decision

1. **Claude children keep the stdin stream-json pipe.** Plain `claude -p` exits
   about 2.5 s after its turn goes idle, even with a mod timer or listener
   process alive, so an injector can never reach an idle child. No mod hook
   restarts a turn, and `$.prompt.submit` waits for the current turn to end. What
   does work under `-p` is Claude's own background-task handling, which
   [D-claude-native-hold](completion-drain.md#d-claude-native-hold) already
   builds on.
2. **A Claude primary is injected through Claude Code's inbox socket.** Meridian's
   `meridian-session-boundary` `SessionStart` hook already runs in every Claude
   launch. It now records `CLAUDE_CODE_MESSAGING_SOCKET`, a documented variable,
   in the session-boundary journal
   ([Claude native sessions](../architecture/claude-native-sessions.md#launch-owned-sessionstart-journal-and-current-chat)).
   `spawn message` posts one SDK stream-json user line,
   `{"type":"user","message":{"role":"user","content":…}}`, to the latest socket
   recorded for the primary's Claude process. Compared with a mod, the inbox:
   - delivers mid-turn, between tool calls, as children get over stdin; a mod's
     `$.prompt.submit` only queues until the turn ends;
   - needs no mod to load, where a mod that fails to load (for example under an
     organisation's plugin policy) fails silently;
   - needs no listener process;
   - brings Claude's own peer framing and inbound controls with it.
3. **Meridian adds no authority of its own.**
   - It never reads or sends `CLAUDE_CODE_MESSAGING_TOKEN`. Meridian is not
     Claude's child; presenting the token claims that it is, which on macOS skips
     Claude's inbound controls. Linux delivers without it.
   - It never wraps a message in a `<cross-session-message from-mode="bypass">`
     envelope. That attestation is undocumented and unearned: any agent could
     then use `spawn message` to steer a bypass-mode session past the hold.
   - It never sets `crossSessionInbound` through `--settings`, which outranks user
     settings and would override a user's own `refuse`.

   The cost is that a primary that bypasses permission prompts holds each
   message behind Claude's approval screen. That covers `--yolo` and a session
   whose Claude settings have `permissions.defaultMode: "auto"`, which Claude's
   gate counts as bypass (the hold cause is `no-mode-asserted`: the sender
   attested no permission mode; gate facts in the
   [research page](../research/claude-code-mods-and-inbox.md#inbound-controls)).
   A human is at every TUI primary, and a user who wants unattended delivery sets
   `crossSessionInbound: "accept"` in their own settings. Meridian documents that
   remedy in meridian-cli `docs/harness-integration.md` and names it in the
   post confirmation, as the owner asked ("can u make sure how to avoid it is
   properly documented"); it never sets the value itself.
4. **"Posted", never "delivered".** The socket writes nothing back; delivered,
   held and refused all look like a closed connection. The CLI reports the
   message as posted and says Claude may hold it. A timeout after the whole line
   was sent also counts as posted, so a retry cannot duplicate the message.
5. **Primary inject is an optional harness port.** `HarnessBundle.primary_inbox`
   sits beside the other optional ports (`primary_idle_sensor`, `detect_ttl`).
   `post_to_primary` in `lib/harness` dispatches by the record's harness and
   raises "<harness> primaries have no message route yet (see #498)" when the port
   is absent. The CLI branches only on `kind == "primary"`; every other kind keeps
   the control-socket path unchanged. A primary gets no wait or retry: a missing
   socket fails at once.
6. **A stale record fails closed.** Claude names its socket after its PID, and a
   spawn row can outlive its Claude process; the PID may then belong to an
   unrelated Claude session. The hook stamps each record with `recorded_at`, and
   the port posts only when the PID is alive, not a zombie, and born no later
   than `recorded_at` (+1 s tolerance). A reused PID is always born after the
   original Claude died, which is after its record was written. "Unknown"
   refuses.

### Rejected

| Alternative | Why rejected |
|---|---|
| A mod replacing the child stdin pipe (the owner's starting direction) | `-p` exits once idle; nothing found holds it open for new input (`spawn:p8410`) |
| A settings-file `Stop` hook returning `block` to hold `-p` | Not probed. Even if it works it is more machinery than stdin: a hook process per idle wait, the hook timeout capping each wait, and Meridian deciding the stop a second time |
| A Meridian mod in the TUI primary: a listener plus `$.prompt.submit` | Works (`spawn:p8415`), but only queues until idle, fails silently when it doesn't load, and adds a process. It remains rejected as the primary inject route. A separate `$.ui.status` spawn row was later built and probed in [PR #706](https://github.com/haowjy/meridian-cli/pull/706), then rejected and closed without merge; it does not change this inject decision. |
| `--messaging-socket-path` | Hidden and undocumented; the documented hook variable finds the socket |
| Storing or sending the messaging token; forging a `from-mode="bypass"` envelope; `crossSessionInbound: accept` through `--settings` | Each gives Meridian authority the user didn't grant (decision 3) |
| Claude Code channels | Research preview, organisation-gated, and a custom server needs `--dangerously-load-development-channels` |
| Remote Control | Routed through the cloud; needs a claude.ai login |
| A `harness == "claude"` branch in the CLI | Harness specifics belong in `lib/harness/` |
| Routing through `SpawnManager` / `ControlActionCoordinator` | A Claude primary has no in-process connection, and faking a sequence number would lie. [D-unified-delivery](#d-unified-delivery) later reached the coordinator another way: it wraps a route, not a connection |
| A `HarnessCapabilities` boolean | The CLI would still own Claude's wire format |
| Persisting the launch gate's `birth_epoch` on the spawn record for the PID check | New spawn-record state for one consumer |
| `is_process_alive(created_after=started_at)` | `started_at` is the spawn's start, not this Claude process's, and its 30 s guard is coarse |

### Consequences

- This route covered Claude alone until [D-unified-delivery](#d-unified-delivery)
  gave every primary harness one.
- `spawn message` cannot tell a user whether a message was held or refused. Claude
  sends status receipts only to a reply socket a real sender names, in an
  undocumented format; the follow-up is in meridian-cli
  `src/meridian/lib/harness/.context/FUTURE`.
- The message schema is read from the Claude Code binary, not the docs. A Claude
  Code release can change it without notice.
- A posted message reaches the model framed as coming from another Claude
  session, not typed by its user. Claude's frame names no Meridian sender;
  [D-spawn-message-frame](#d-spawn-message-frame) adds one in the text.

**Revisit when:** the owner rules on the flag above, or Claude Code documents the
inbox message schema or its receipts, or changes its inbound controls.

**Provenance:** `work:claude-mod-inject` `decisions.md` D1–D10,
`probe-mod-inject.md` (`spawn:p8410`), `probe-tui-mod.md` (`spawn:p8415`),
`probe-inbox-socket.md` (`spawn:p8419`), `verification-review-fold.md`; research
`spawn:p8416`; seam map `spawn:p8422`; reviewers `spawn:p8430`, `spawn:p8431`,
`spawn:p8441`; owner direction `chat:c8112`; session mining `spawn:p8443`.

---

## D-spawn-message-frame: every message names its sender in text, and offers a reply only when one works (2026-10) {#d-spawn-message-frame}

**Status:** merged in PR #777 (`5ad479b0a`), with the rename of `spawn inject`
to `spawn message`. Current choice: the owner asked for a reply id; the
frame's wording and the reply-route rule are the implementer's. The chat-ID reply
address and the cross-project reply below shipped in meridian-cli v0.16.4 (#801),
with [D-spawn-message-address](#d-spawn-message-address). Naming the sender's
current chat rather than its entry chat shipped in meridian-cli v0.17.1 (#811). The code contract is
in meridian-cli `src/meridian/cli/.context/CONTEXT.md` ("Injected message
frame").

### Problem

A session that received a message could not tell which session sent it or how
to answer. Claude's peer framing says only "another Claude session"; the
control-socket route to a child carries the text alone, with no sender id or
reply address. The owner: "there is no id for the
response... so we should probably add like 'this was sent from a meridian
session. use `meridian spawn message c9297 etc...'".

### Decision

1. **Provenance is message text.** The CLI prepends one line before posting:
   `[Meridian] Message from spawn pX (chat cY). To answer, run`
   `` `meridian spawn message cY --message "..."`. `` The reply names the
   sender's current chat (`continue_chat_id`), because a primary gets a new spawn
   ID on each `--continue` while its chat ID stays, and a `/clear` moves it to a
   new chat; `spawn message` refuses the chat it left. A sender in another project gets `-C <sender project root>` in the
   command. Text is the one thing every route
   carries unchanged and every harness renders, so the frame is built once and reads
   the same everywhere.
2. **Advertise a reply route only when it works.** The reply command appears only when
   the sender's record is active and its control socket file exists, the same file
   the post looks for. Anyone else, a finished spawn, a primary with no route, or an unknown
   id gets "it has no reply route here"; a message with no sender identity says it has
   no reply address. Without the check, a child told to answer a `--bare` Claude primary
   would run a command that exits 1. The check does not connect, so a dead socket on a
   live runner still gets the line, a Codex or OpenCode primary whose TUI moved still
   gets it and then refuses, and a child still binding its socket is framed as having
   no route.
3. **The chat id comes from the sender's own record**, not `$MERIDIAN_CHAT_ID`,
   which names the conversation that launched the sender and would point a
   reply at the wrong session.

### Rejected

| Alternative | Why rejected |
|---|---|
| Claude's peer envelope `from-mode` | An attestation Meridian has never forged; an agent could steer a bypass-mode session past review ([decision 3](#d-claude-primary-inbox)) |
| Claude's envelope `from` alone | Claude-only; the frame must read the same in every harness |
| Always printing the reply command | Points children at routes that fail |

**Revisit when:** a harness carries a sender field through its route.

**Provenance:** decision record `spawn:p9612` `decisions-spawn-message.md`
D1–D4; owner requests in chat `c9331`; PR #777.

---

## D-spawn-message-address: one address form for any live session, in this project or another (2026-10) {#d-spawn-message-address}

**Status:** current choice (tech lead), answering the owner's request: "we need
an easier way to do cross/repo communication... (and learn how to do it), and we
need to make sure `meridian spawn message` is much more easily accessible".
Shipped in meridian-cli v0.16.4 (#801). User-facing behavior: meridian-cli `docs/commands.md`; how an agent
uses it: the bundled `messaging__meridian` skill.

### Problem

Two interactive sessions in different repos needed to coordinate: `chat:c9618`
in meridian-cli and `chat:c14995` in meridian-flow. Neither found
`spawn message`: it was missing from `meridian -h`, and its help framed it as a
way to forward the user's direction to a child. One fell back to Claude's own
`SendMessage` through tmux. The command also rejected chat IDs, though sessions
know each other by chat ID, and a message sent with `-C` into another project
arrived as "no sender identity; it has no reply address".

### Decision

1. **The address is `meridian -C <project root> spawn message <chat ID>`.** Chat
   and spawn IDs are numbered per project, so a bare ID is ambiguous across
   repos; the project root makes it exact. Given a chat ID, `spawn message` sends to
   the run live in that chat now, which can differ from the chat's owner
   ([D-chat-names-own-session](chat-spawn-ownership.md#d-chat-names-own-session)),
   and refuses a chat with no live run. Since v0.17.1 (#811),
   `show` and `cancel` name that live run too
   ([D-locate-projections](ref-classification.md#d-locate-projections)).
2. **Every session advertises its own address, and every message carries the
   sender's.** Every primary shows its address in its capabilities block, as one
   command another session can paste, whether or not that launch ends up with a route:
   a `--bare` Claude primary states an address that leads to the no-route error, because
   the launch can't know before Claude starts
   ([a socket file means a route](#a-socket-file-means-a-route)). The frame
   ([D-spawn-message-frame](#d-spawn-message-frame)) carries the reply command,
   with `-C` only when the sender is in another project.
3. **The `-C` identity drop (#794) stays.** `-C` into another project still drops
   the caller's project-scoped `MERIDIAN_*` IDs before acting, so nothing the
   command launches or writes there inherits them. The CLI reads the caller's
   spawn ID and project root just before the drop and carries them in
   `GlobalOptions.caller`, never back into the environment. Only `spawn message`
   reads it, to build the frame. The sender's record is read without creating
   state in the sender's project.
4. **Findable where agents look.** `spawn message` is listed in `meridian -h`;
   every ref-taking command's help lists the ref forms it accepts; a primary-only
   capabilities line says how to message; the bundled `messaging__meridian` skill
   covers when to message, how to address, and how to reply. The owner: "it isn't
   default in llm's knowledge despite how imporatnt it kinda is".

### Rejected

| Alternative | Why rejected |
|---|---|
| Bare chat IDs as global addresses | IDs are counted per project and collide across repos |
| A project-qualified ref syntax (for example `project:cN`) | Needs a project-name registry Meridian does not have; possible follow-up |
| Restoring the caller's IDs into the env under `-C` | Re-opens #794: children launched in the other project would claim the caller's identity |
| The spawn ID as the reply address | A primary's spawn ID changes with each `--continue`; its chat ID does not |

**Revisit when:** Meridian gains a project registry, or a harness carries a
native sender address that survives `-C`.

**Provenance:** `chat:c9677` (owner requests, lead decisions); trigger sessions
`chat:c9618` and meridian-flow `chat:c14995`; reviews `spawn:p9988`,
`spawn:p9992`.

---

## D-delivery-kind: the sender is told how far a message got (2026-11) {#d-delivery-kind}

**Status:** merged in PR #878 (v0.19.6). Tech-lead choice; the owner has not ruled on the
wording. The contract per harness is in meridian-cli
`src/meridian/lib/harness/.context/CONTEXT.md` ("Inject Acknowledgment").

`send_user_message` returns a kind, and the kind travels to the sender. `acknowledged`
means the harness answered the request that carried the message, and the CLI prints
"Message delivered". `posted` means the write finished and nothing confirmed it, and the
CLI prints "Message posted … (no receipt)". The connection states the kind because only it
knows whether it waited for an answer. Neither word claims the model read the message.

**Why a return value.** A static capability flag would have been smaller at the time, but
[D-unified-delivery](#d-unified-delivery) needed a delivery port that returns this same
shape, so a flag would have been replaced by that work and the return value was extended
by it. Inferring the kind from the harness id in `SpawnManager` was rejected because harness
specifics stay in `lib/harness/`.

**A missing field reads as `posted`.** A launcher from an older build does not say how far
the message got, so the CLI claims no more than it was told. The word is frozen: renaming
`acknowledged` on the wire would downgrade older launchers' replies to "posted".

**"Delivered" on a Codex steer stays.** A message sent during a turn is acknowledged when
the app-server accepts `turn/steer`, before the text is in the rollout and before the model
has used it. A softer sentence was proposed. The word stayed because in every run the
acknowledged steer reached the rollout within about 2 s, survived a cancel, and was read at
the model's next step ([what a steer does](../research/reaching-a-live-tui.md#codex)). A
`session log` taken the moment the send returns can still lack the text.

**OpenCode V1** is marked `acknowledged` from reading its code only; no V1 server was run.

---

## D-no-retry-after-write: a message is resent only if nothing was written (2026-11) {#d-no-retry-after-write}

**Status:** merged in PR #877 (v0.19.6), then reshaped by PR #907. Tech-lead choice. The
rule for connection authors is in meridian-cli
`src/meridian/lib/harness/connections/AGENTS.md`.

`ControlActionCoordinator.run_action` retries once on `DeliveryRefused` and nothing else.
Codex `turn/start` waits for its JSON-RPC response after the write, so a slow reply raises
`TimeoutError` with the turn already started, and retrying that starts a second turn. A
fake connection that writes and then times out reached the harness twice before this
change and once after. No real Codex timeout was forced, so the duplicate turn itself is
inferred.

**Three outcomes, named where the failure happens.** `DeliveryRefused` means the receiver
is alive and nothing was written. `TransportDead` means the receiver is gone and nothing
was written. Anything else may have followed the write, is never resent, and tells the
sender the message "may have arrived and was not resent". The first version had one error,
`ConnectionNotReady`, which covered a dead process and a session still starting alike, so
the coordinator had to guess. Only the code at the raise site knows, so the class is
chosen there. The default is the safe direction: a raise nobody classified over-warns the
sender and never claims delivery, so a misclassified raise is cleanup, not a redesign.

Sends are bounded at 33 s (`SEND_TIMEOUT_SECONDS`), above the harnesses' own 30 s timeouts,
whose text should win, and below the CLI's 35 s wait, so a hung send can't block a stop. The
completion hold reads the same classes (meridian-cli
`src/meridian/lib/streaming/.context/completion-and-entry-points.md`).

**Rejected:** an idempotency key per message, because Codex and Claude have no way to
deduplicate a turn; translating timeouts in each connection, because only the coordinator
knows the failure followed a journalled send.

**Cost:** a transient failure on an interrupt is now reported once. A repeated interrupt
after a failed one is a separate bug, #874.

---

## D-unified-delivery: every live session serves one control socket (2026-11) {#d-unified-delivery}

**Status:** built on `release/v0.20.0` through PRs #907, #917 and #918, for release as
v0.20.0. The owner settled the shape and each call below on 2026-11-06, except the
refusal after a TUI session switch, which has [its own record](#d-primary-session-move).
Remaining edges: [#916](https://github.com/haowjy/meridian-cli/issues/916). How each harness
behaves: meridian-cli `docs/commands.md` and `docs/harness-integration.md`.

### Problem

`spawn message` had two transports chosen by `kind == "primary"`: the control socket for
children, and a CLI-side post into Claude's inbox. Adding Codex, OpenCode and Pi on that
structure would have given each its own route and its own check of whether a reply could
reach it. The check and the post had already disagreed: a Claude primary with no published
inbox was offered a reply command that failed.

### Choice

**One socket, one wire format, one CLI path.** `ControlSocketServer` takes a `MessageRoute`
instead of a `SpawnManager`, and the sender neither knows nor asks what kind of session it
reaches. What stands behind the socket differs:

| Session | Who serves the socket | Where a message goes |
|---|---|---|
| Spawned child | Its `SpawnManager` | The harness connection |
| Claude primary | The PTY launcher, on a daemon thread | Claude's inbox |
| Codex, OpenCode primary | `PrimaryAttachLauncher` | The observer connection it already holds |
| Pi primary | The `meridian-message` extension, inside Pi | `pi.sendUserMessage` |

Every server but Pi's journals through `ControlActionCoordinator`, so primaries gained
`control_actions.jsonl`. Pi's extension runs in another language and process and does not.

**The observer became load-bearing.** It was built to watch, and it was harmless for it to
lag, die after 120 s of silence, or stay on its first session. As a route it has to stay
connected through any idle silence and notice when the TUI leaves. Meridian never answers an
approval an injected turn raises: approvals reach every Codex client, so an answer from
Meridian would beat the human to it.

**The Pi extension serves the socket itself**, at the same path and wire format, because
Pi has no external channel and something inside its process must listen. It is a new
extension because Meridian's Pi extensions each own one concern, and messaging is not idle
tracking. The launcher computes the path and passes it by env var, so the hash stays in one
language. The cost is a TypeScript copy of the request and response handling: a change to
one side needs the other.

**Unused socket verbs went.** `interrupt`, `permission_reply` and `user_input_reply` had no
client.

### A socket file means a route

The launcher never predicts
whether a session can take messages. It asks the harness every 0.25 s and binds only once
the harness offers a route. Prediction was the alternative, and it would have to know
`--bare`, `--safe-mode`, `--print`, Claude's version, a pending folder-trust prompt that
delays the inbox, and whatever comes next. Waiting for the fact covers all of them, and it
makes the file the one thing the frame's reply check and the post both consult, so they can
no longer disagree. Two limits: it says a route exists, not that it will accept a
message ([a moved TUI refuses](#d-primary-session-move)), and a Pi primary launched with a
prompt has no file until its first `agent_start`. The CLI's 30 s poll covers that gap.

### Calls settled on 2026-11-06

| Question | Settled | Why, and what it costs |
|---|---|---|
| A primary from an older build | No route until relaunched, no fallback code | Old sessions are rarely resumed; the owner carries work forward with `--from` or session mining. The sender waits 30 s, then gets an error that says to relaunch |
| OpenCode's idle sensor sees an injected turn as a human prompt | Accepted | It counts as activity, as a typed prompt does |
| A message to a busy Codex primary | Steers into the current turn | It matches what `send_user_message` already does for children, and the message reaches the model at its next step |
| Claude's peer-safety paragraph | Accepted | About 130 tokens on each inbox message ([measured](../research/claude-code-mods-and-inbox.md#the-inbox-socket)). No setting removes it, and only Claude primaries pay it |
| Which release | `release/v0.20.0` | Ahead of the v0.21 first proposed |

### Rejected

| Alternative | Why rejected |
|---|---|
| Per-harness `primary_inbox` ports behind the CLI branch | Keeps the `kind` branch and a reachability rule per harness |
| A library resolver returning `ControlSocket`, `NativeInbox` or `Unreachable`, with the CLI posting | Managed primaries need a launcher socket anyway: only the launcher holds the observer, and OpenCode's attach password is never on disk. A second transport would keep the branch forever |
| Pi: the launcher serves and forwards to an extension socket | A second, Pi-only socket for no gain. The worry behind it was false: the extension can reach the path |
| Pi: a file drop | Gives the sender no failure |
| One PR for everything | The refactor could land before the routes it enables |

### Costs

A primary with no route takes the 30 s discovery wait instead of failing at once, which
includes a `--bare` Claude primary that used to fail in about a second.

### What was run

At the merge of #918, real Claude, Codex, OpenCode and Pi primaries each took an idle
message and rendered it, Pi took one mid-turn and OpenCode one mid-turn, a Pi child and a Pi
primary exchanged messages both ways, and Codex and OpenCode refused after a move. An older
build's primary and a `--bare` Claude primary both got the no-route error. Not yet run there: mid-turn on Claude and Codex primaries, a Codex primary after more than three idle
minutes, full child round trips on Claude, Codex and OpenCode, and a message sent immediately
after launch.

---

## D-primary-session-move: a Codex or OpenCode route refuses once the TUI leaves its session (2026-11) {#d-primary-session-move}

**Status:** built in PR #917. The tech lead's call, reported to the owner and not yet
answered. Following the TUI stays open as a follow-up in
[#916](https://github.com/haowjy/meridian-cli/issues/916).

### Problem

A Claude or Pi route lives inside the TUI process, so it follows whatever session the human
opens. A Codex or OpenCode route is a second client of a server, on the session the
launcher started. After `/new` in the TUI, a message was reported delivered and its turn
ran on the abandoned session: it wrote files and spent tokens, the human saw nothing, and an
approval it raised stayed pending where no prompt could reach them.

### Choice

The route stays on the first session and refuses, with `DeliveryRefused` and a message
saying why, once the launcher sees the TUI leave. The refusal is sticky for the launch,
including after the human returns to the first session, and it changes nothing else: the
socket stays bound and the sender can tell the human.

A false "delivered" is the outcome this exists to end, and following can't end it. No server
reports which session a TUI shows ([what the wire shows](../research/reaching-a-live-tui.md#which-session-is-the-tui-on)).
A Codex `/resume` to a loaded thread and an OpenCode switch to an existing session give no
usable signal, and a new Codex thread can't be resumed until its rollout exists. The rest of
Meridian (`harness_session_id`, the idle sensor, `--continue`) stays on the first session
as well, so following the route alone would split Meridian's view of the session.

### Rejected

| Alternative | Why rejected |
|---|---|
| Follow `/new` and `/fork` only, refuse the rest | More code for partial cover, and it splits Meridian's view of the session |
| Keep delivering to the first session | The false "delivered" above |
| Treat a loaded-thread list or `thread/closed` as focus | Neither means that: several threads stay loaded, and `thread/closed` means unloaded |

### What it leaves

An OpenCode message sent after the new-session or switch-session command, before the human
types there, goes to the first session and reports delivered, because the server sends
nothing until then. A misread helper thread would refuse for the rest of the session; that
fails honestly, so detection leans toward refusing. The signals and the list of gaps are in
meridian-cli `docs/harness-integration.md`.

**Revisit when:** the owner answers; a harness reports its TUI's focus; or `--continue`,
`harness_session_id` and the idle sensor are made to follow the TUI.
