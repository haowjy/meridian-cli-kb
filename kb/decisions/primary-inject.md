# Decisions: Primary Inject

How `meridian spawn message` reaches a **primary**: a session a human started in a
harness TUI, how every message names its sender
([D-spawn-message-frame](#d-spawn-message-frame)), how one session addresses
another, in this project or another
([D-spawn-message-address](#d-spawn-message-address)), and how the sender learns what
happened to its message ([D-delivery-kind](#d-delivery-kind),
[D-no-retry-after-write](#d-no-retry-after-write)). The plan to give every primary a
route is [D-unified-delivery](#d-unified-delivery). A spawned child is injected through its `control.sock`, which the
child's `SpawnManager` serves over the harness connection. A primary runs the
harness TUI directly and has no `SpawnManager`, so nothing serves that socket.
The code contract for the route below is in meridian-cli
`src/meridian/lib/harness/.context/CONTEXT.md` ("Primary inbox injection"). The
Claude Code behaviour it depends on is in
[Claude Code: mods and the inbox](../research/claude-code-mods-and-inbox.md).
The command was called `spawn inject` until PR #777; that spelling remains a
hidden alias that warns, for one release (removal tracked in meridian-cli
`src/meridian/cli/.context/TODO`). "Inject" still names the mechanism: the Pi
control action, `SpawnManager.inject`, and primary inject as a concept.

---

## D-claude-primary-inbox: a Claude primary is injected through Claude Code's own inbox socket, not a mod (2026-10) {#d-claude-primary-inbox}

**Status:** decided by the tech lead on probe evidence (`work:claude-mod-inject`
D2, D6–D9). Issue #665; merged in PR #668 (v0.10.22). Before it, the command
against any primary polled 30 s for a control socket that never appeared, then
failed with `spawn not running: pN has no control endpoint`.

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
| Routing through `SpawnManager` / `ControlActionCoordinator` | A Claude primary has no in-process connection; faking `inbound_seq` would lie. Codex and OpenCode primaries do hold one, which [D-unified-delivery](#d-unified-delivery) builds on |
| A `HarnessCapabilities` boolean | The CLI would still own Claude's wire format |
| Persisting the launch gate's `birth_epoch` on the spawn record for the PID check | New spawn-record state for one consumer |
| `is_process_alive(created_after=started_at)` | `started_at` is the spawn's start, not this Claude process's, and its 30 s guard is coarse |

### Consequences

- Codex, OpenCode and Pi primaries still have no message route; they fail at once
  with the #498 message
  ([future work](../open-questions/future-work.md#live-primary-injection-control-endpoint-498)).
  [D-unified-delivery](#d-unified-delivery) would replace this port rather than fill
  it per harness.
- `spawn message` cannot tell a user whether a message was held or refused. Claude
  sends status receipts only to a reply socket a real sender names, in an
  undocumented format; the follow-up is in meridian-cli
  `src/meridian/lib/harness/.context/FUTURE`.
- The message schema is read from the Claude Code binary, not the docs. A Claude
  Code release can change it without notice.
- A posted message reaches the model framed as coming from another Claude
  session, not typed by its user. Claude's frame names no Meridian sender;
  [D-spawn-message-frame](#d-spawn-message-frame) adds one in the text.

**Revisit when:** the owner rules on the flag above; Claude Code documents the
inbox message schema or its receipts, or changes its inbound controls; or Codex,
OpenCode or Pi gains a native input route for a running TUI.

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
   command. Text is the one thing both
   routes (child control socket, Claude primary inbox) carry unchanged and every
   harness renders, so the frame is built once and reads the same everywhere.
2. **Advertise a reply route only when it works.** The reply command appears
   only when the sender's record is active and Meridian can post to it: a
   spawned child, or a primary whose harness bundle implements the optional
   `primary_inbox` port (the harness's route into a running TUI). A Codex or
   OpenCode primary, a finished spawn, or an unknown id gets "it has no reply
   route here" instead; a message with no sender identity says it has no reply
   address. Without the check, a child
   told to answer an OpenCode primary would run a command that exits 1.

   The check is coarser than the post. It asks whether the harness registers an
   inbox and whether the sender is a child, while the post also needs a published
   inbox socket, a live process, or a child's control socket. So a `--bare` Claude
   primary, or a child with no socket, is offered a reply that fails. Under review
   on branch `fix/reply-route` (no PR yet, not merged): the frame runs the post's
   own resolution, and a child needs its socket. It does not connect, so a dead
   socket on a live runner still gets the line, and a child still binding its
   socket is framed as having no route.
3. **The chat id comes from the sender's own record**, not `$MERIDIAN_CHAT_ID`,
   which names the conversation that launched the sender and would point a
   reply at the wrong session.

### Rejected

| Alternative | Why rejected |
|---|---|
| Claude's peer envelope `from-mode` | An attestation Meridian has never forged; an agent could steer a bypass-mode session past review ([decision 3](#d-claude-primary-inbox)) |
| Claude's envelope `from` alone | Claude-only; the frame must read the same in every harness |
| Always printing the reply command | Points children at routes that fail |

**Revisit when:** a harness carries a sender field through both routes, or
Codex, OpenCode or Pi primaries gain a message route (their senders then get
the reply command through the same port).

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
   sender's.** A primary whose harness has a message route shows its address in
   its capabilities block, as one command another session can paste. The frame
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

## D-delivery-kind: the sender is told how far a message got (2026-11, in review) {#d-delivery-kind}

**Status:** in review as PR #878 (`fix/message-receipt`), not merged. Tech-lead
choice; the owner has not ruled on the wording. Until it merges, `spawn message`
prints "Message delivered" for any successful response, including a Claude child's
stdin write that nothing confirmed. The contract, per harness, is in meridian-cli
`src/meridian/lib/harness/.context/CONTEXT.md` ("Inject Acknowledgment") on that
branch; what follows is what that page does not say.

`send_user_message` returns a kind, `posted` or `acknowledged`, and the kind travels
to the sender. It is stated by the connection because only the connection knows
whether it waited for a harness answer. Neither word claims the model read the
message.

**Why a return value.** A static capability flag would be smaller today, but
[D-unified-delivery](#d-unified-delivery) needs a delivery port that returns this same
shape, so a flag would be replaced by that work and the return value extended by it.
Inferring the kind from the harness id in `SpawnManager` was rejected because harness
specifics stay in `lib/harness/`.

**A missing field reads as `posted`.** A launcher from an older build does not say
how far the message got, so the CLI claims no more than it was told. This freezes the
word: once the PR ships, renaming `acknowledged` on the wire would downgrade older
launchers' replies to "posted".

**This does not change decision 4 of
[D-claude-primary-inbox](#d-claude-primary-inbox).** The Claude inbox writes nothing
back, so it stays posted. A Pi primary could only report posted too, because
`sendUserMessage` returns nothing
([research](../research/reaching-a-live-tui.md#pi)).

**What was run.** Against a build merging the patch branches, Claude children printed
posted, and Codex, OpenCode 2.x and Pi children printed delivered (`spawn:p10517`).
OpenCode V1 is marked `acknowledged` from reading its code only, and the `posted`
journal status was added after those runs.

---

## D-no-retry-after-write: a control action is retried only if nothing was written (2026-11, in review) {#d-no-retry-after-write}

**Status:** in review as PR #877 (`fix/no-retry-after-write`), not merged. Tech-lead
choice. Until it merges, `ControlActionCoordinator` also retries `TimeoutError`,
`ConnectionError` and `OSError`.

`ControlActionCoordinator.run_action` retries `ConnectionNotReady` and nothing else.
That is the one error every connection raises before writing. Codex `turn/start`
waits for its JSON-RPC response after the write, so a slow reply raises `TimeoutError`
with the turn already started, and retrying that starts a second turn. A fake
connection that writes and then times out reached the harness twice before this
change and once after. No real Codex timeout was forced, so the duplicate turn itself
is inferred.

The failure text says so: "timed out after the request was sent; it may have been
delivered and was not resent". `spawn message` waits 35 s, longer than the 30 s Codex
and Pi request timeouts, so that text usually reaches the sender.

**Rejected:** an idempotency key per message, because Codex and Claude have no way to
deduplicate a turn; translating timeouts in each connection, because only the
coordinator knows the failure followed a journalled send.

**Cost:** the coordinator also carries interrupt and the permission and user-input
replies, so a transient failure on one of those is now reported once. Nothing in
Meridian's CLI sends the two reply verbs. A repeated interrupt after a failed one is a
separate bug, #874.

---

## D-unified-delivery: every live launcher serves one control socket (2026-10, not built) {#d-unified-delivery}

**Status:** current choice, not built, and not confirmed for building. None of this
exists in any branch. The owner's whole input was three clicks on the lead's
recommended options (`chat:c9424`, 2026-10-26): "Unified socket (Recommended)", "No
route until relaunch (Recommended)", "Refactor PR, then routes (Recommended)". The
sentences in the work item's design notes are the lead's. On 2026-11-06 the lead told
the owner this design "was never settled" and would be checked with them before
building (`chat:c10170`). A confirmation request is in `work:native-primary-message-routes`
`u-recommendation.md`; the owner has not answered.

### Problem

`spawn message` has two transports chosen by `kind == "primary"`: the control socket
for children, and a CLI-side post into Claude's inbox. Adding Codex, OpenCode and Pi
on that structure adds a third route and a second reachability rule, and each harness
inherits the bugs in the reply check and the retry.

### Choice

1. `ControlSocketServer` takes a delivery port instead of a `SpawnManager`. Every live
   launcher serves it: `SpawnManager` for children, the connection the launcher already
   holds for Codex and OpenCode primaries, and a sink in the PTY launcher for Claude
   (forwarding to the inbox) and Pi.
2. The CLI keeps one path. `post_to_primary`, `primary_accepts_messages`, the
   `primary_inbox` port, the `kind == "primary"` branch and the PID-reuse guard go,
   because a live launcher owns its child and that PID cannot be reused while it serves.
3. A primary launched by an older build serves no socket and has no route until it is
   relaunched. No fallback code.
4. A refactor PR comes first (the port, the dead socket verbs, Claude onto the launcher
   socket), then Codex and OpenCode, then Pi.

Probes removed the two worries that could have overturned this. The Codex observer
survives TUI attach, so no reconnect is needed, and approvals reach the TUI, so an
injected turn does not stall on a request only Meridian can see
([research](../research/reaching-a-live-tui.md#codex)). Earlier notes assumed the
opposite of both.

### Rejected

| Alternative | Why rejected |
|---|---|
| Per-harness `primary_inbox` ports behind the CLI branch | Keeps the `kind` branch and a reachability rule per harness |
| A library resolver returning `ControlSocket`, `NativeInbox` or `Unreachable`, with the CLI posting | Managed primaries need a launcher socket anyway: only the launcher holds the observer, and OpenCode's attach password is never on disk. A second transport would keep the branch forever |
| A one-release fallback for older primaries | Primaries are short-lived |
| One PR for everything | The refactor can land before the routes it enables |

### Costs

A socket-serving thread in the synchronous PTY launcher, which has no event loop.
Primaries get the 30 s discovery poll instead of failing at once. A Claude process
whose launcher died loses its route.

### Open for the owner

- **Pi:** the launcher serves the socket and forwards to an extension listener (two
  hops, keeps "the launcher serves" true), or the extension serves it (one hop, needs a
  new variable for the runtime root). The lead's recommendation moved from extension to
  launcher.
- **OpenCode:** an injected turn reads as a human prompt to the idle sensor, so it
  counts as activity. The lead recommends accepting that.
- **Codex mid-turn:** steer into the TUI's turn, as `send_user_message` does today. The
  lead recommends this.
- **The port:** `deliver(text) -> DeliveryKind` where any failure raises, which removes
  `InjectResult`, `on_result` and `inbound_seq`. A review proposed it (`spawn:p10531`).

**Provenance:** `work:native-primary-message-routes` `design-notes.md`,
`u-recommendation.md`, `explore-unified-delivery.md`; structural review
`spawn:p9722`; probes `spawn:p9741`, `spawn:p9736`; session mining `spawn:p10549`.
