# Decisions: Primary Inject

How `meridian spawn message` reaches a **primary**: a session a human started in a
harness TUI, how every message names its sender
([D-spawn-message-frame](#d-spawn-message-frame)), and how one session addresses
another, in this project or another
([D-spawn-message-address](#d-spawn-message-address)). A spawned child is injected through its `control.sock`, which the
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
| Routing through `SpawnManager` / `ControlActionCoordinator` | A primary has no in-process connection; faking `inbound_seq` would lie |
| A `HarnessCapabilities` boolean | The CLI would still own Claude's wire format |
| Persisting the launch gate's `birth_epoch` on the spawn record for the PID check | New spawn-record state for one consumer |
| `is_process_alive(created_after=started_at)` | `started_at` is the spawn's start, not this Claude process's, and its 30 s guard is coarse |

### Consequences

- Codex, OpenCode and Pi primaries still have no message route; they fail at once
  with the #498 message. Each fills the same port when it gains one
  ([future work](../open-questions/future-work.md#live-primary-injection-control-endpoint-498)).
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
address and the cross-project reply below are on meridian-cli branch
`feat/spawn-message-peers` (PR not yet opened), with
[D-spawn-message-address](#d-spawn-message-address). The code contract is
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
   `` `meridian spawn message cY --message "..."`. `` The reply names the chat,
   because a primary gets a new spawn ID on each `--continue` while its chat ID
   stays. A sender in another project gets `-C <sender project root>` in the
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
   `primary_inbox()` in `lib/harness/bundle.py` is the one lookup behind both
   `post_to_primary()` and the reachability check `primary_accepts_messages()`,
   so the frame and the post cannot disagree.
3. **The chat id comes from the sender's own record**, not `$MERIDIAN_CHAT_ID`,
   which names the root conversation and would point a reply at the wrong
   session.

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
Built on meridian-cli branch `feat/spawn-message-peers` (PR not yet opened); not
on `main`. User-facing behavior: meridian-cli `docs/commands.md`; how an agent
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
   repos; the project root makes it exact. A chat ID reaches the run live in that
   chat now ([D-chat-names-own-session](chat-spawn-ownership.md#d-chat-names-own-session)).
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
