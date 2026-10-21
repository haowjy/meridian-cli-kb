# Decisions: Primary Inject

How `meridian spawn inject` reaches a **primary**: a session a human started in a
harness TUI. A spawned child is injected through its `control.sock`, which the
child's `SpawnManager` serves over the harness connection. A primary runs the
harness TUI directly and has no `SpawnManager`, so nothing serves that socket.
The code contract for the route below is in meridian-cli
`src/meridian/lib/harness/.context/CONTEXT.md` ("Primary inbox injection"). The
Claude Code behaviour it depends on is in
[Claude Code: mods and the inbox](../research/claude-code-mods-and-inbox.md).

---

## D-claude-primary-inbox: a Claude primary is injected through Claude Code's own inbox socket, not a mod (2026-10) {#d-claude-primary-inbox}

**Status:** decided by the tech lead on probe evidence (`work:claude-mod-inject`
D2, D6–D9). Issue #665; merged in PR #668 (v0.10.22). Before it, `spawn inject`
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
`spawn inject` against one waited 30 s and then reported a live primary as "not
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
   `spawn inject` posts one SDK stream-json user line,
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
     then use `spawn inject` to steer a bypass-mode session past the hold.
   - It never sets `crossSessionInbound` through `--settings`, which outranks user
     settings and would override a user's own `refuse`.

   The cost is that a primary launched with `--yolo` (bypass permissions) holds
   each inject behind Claude's approval screen. A human is at every TUI primary,
   and a user who wants unattended delivery sets `crossSessionInbound: "accept"`
   in their own settings.
4. **"Posted", never "delivered".** The socket writes nothing back; delivered,
   held and refused all look like a closed connection. The CLI reports the
   message as posted and says Claude may hold it. A timeout after the whole line
   was sent also counts as posted, so a retry cannot duplicate the message.
5. **Primary inject is an optional harness port.** `HarnessBundle.primary_inbox`
   sits beside the other optional ports (`primary_idle_sensor`, `detect_ttl`).
   `post_to_primary` in `lib/harness` dispatches by the record's harness and
   raises "<harness> primaries have no inject route yet (see #498)" when the port
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

- Codex, OpenCode and Pi primaries still have no inject route; they fail at once
  with the #498 message. Each fills the same port when it gains one
  ([future work](../open-questions/future-work.md#live-primary-injection-control-endpoint-498)).
- `spawn inject` cannot tell a user whether a message was held or refused. Claude
  sends status receipts only to a reply socket a real sender names, in an
  undocumented format; the follow-up is in meridian-cli
  `src/meridian/lib/harness/.context/FUTURE`.
- The message schema is read from the Claude Code binary, not the docs. A Claude
  Code release can change it without notice.
- A posted message reaches the model framed as coming from another Claude
  session, not typed by its user.

**Revisit when:** the owner rules on the flag above; Claude Code documents the
inbox message schema or its receipts, or changes its inbound controls; or Codex,
OpenCode or Pi gains a native input route for a running TUI.

**Provenance:** `work:claude-mod-inject` `decisions.md` D1–D10,
`probe-mod-inject.md` (`spawn:p8410`), `probe-tui-mod.md` (`spawn:p8415`),
`probe-inbox-socket.md` (`spawn:p8419`), `verification-review-fold.md`; research
`spawn:p8416`; seam map `spawn:p8422`; reviewers `spawn:p8430`, `spawn:p8431`,
`spawn:p8441`; owner direction `chat:c8112`; session mining `spawn:p8443`.
