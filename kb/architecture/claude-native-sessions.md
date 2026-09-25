# Architecture: Claude Native Sessions

Claude Code keeps every conversation as one JSONL transcript in a directory that all
Claude processes with the same config root share. This page explains how Meridian
keeps chats pinned to the right transcript in that shared store. It covers the
store layout, how a resume or fork source reaches the child, and how the TUI
trampoline is handled. The cross-harness rule is the
[native session identity decision](../decisions/native-session-identity.md). The
shared seams are in [native session binding](native-session-binding.md). The Claude
identity work described here is on `fix/native-session-wrapper` (draft PR #520), not
on `main`.

## Store layout and the shared-store hazard

```
<config-root>/projects/<slug>/<session-id>.jsonl
```

`<config-root>` is the child's `CLAUDE_CONFIG_DIR` (`~` and relative values resolve
against the child's `HOME` and cwd), else `$HOME/.claude`. `<slug>` is the resolved
working directory with every non-alphanumeric character replaced by `-`. The
Meridian native store for a Claude chat is the resolved
`<config-root>/projects/<slug>` directory
(`ClaudeAdapter.native_store_for_launch`).

Every Claude process sharing a config root shares that store, and Claude's own
recovery is inference-shaped:

1. If Claude loses its session ID, it takes the most recent `.jsonl` in
   `projects/<slug>/`. Two concurrent sessions in the same project can swap.
2. Slug matching can associate `my-project-v2/` with `my-project/`.
3. A nested `meridian spawn` inherits the parent's `CLAUDE_CONFIG_DIR`, so parent and
   child transcripts land in the same namespace.

Meridian does not isolate the config directory per spawn. It avoids relying on any
of these behaviors: every launch names the exact session, and every read opens one
exact file.

## Operations

| Operation | Emitted | Identity |
|---|---|---|
| Create | `--session-id <uuid>` (from `NativeIdentity.session_id`, minted by `assign_session_id`) | Minted and bound before exec |
| Resume | `--resume <id>` | Source verified and seeded before exec |
| Fork | `--resume <id> --fork-session` | New ID unknown until Claude reports it; the first owned identity event binds ID and store together |

Raw passthrough session-identity flags are refused on tracked launches. Owned
identity comes only from qualified Claude envelopes. Session IDs inside nested
message content are never read as identity.

## Source preparation

A resume or fork source is the source chat's recorded `(native_store, id)`
(`SessionRequest.source_native_store`). Before exec, Claude preparation
(`harness/claude_preflight.py:ensure_claude_session_accessible`) opens exactly
`<recorded store>/<id>.jsonl`:

- It validates the file's first line (`validate_claude_session_file`): its
  `sessionId` must equal the ID. A missing, empty, torn, or malformed first line, or
  an ID that is not a plain file name, refuses with
  `NativeSessionUnavailable(missing)`. A readable `sessionId` naming another session
  refuses with `NativeEntryMismatch`. It never searches the ambient config root,
  `~/.claude`, or other projects.
- Otherwise it places the file into the child's own store,
  `<child config root>/projects/<child slug>/<id>.jsonl`. When source and child share
  a config root that is a symlink, created under a unique temporary sibling name and
  moved into place with `os.replace`. Otherwise it is a streamed copy published
  atomically. Either way a crash or error leaves the previous target intact (the
  first version unlinked the target before creating the symlink). If the target
  already is the source, nothing happens.
- The child's effective config root is recorded on the spawn/session
  (`claude_config_dir`) for later reads.

**Why no fallback.** This path once took a config-root hint, re-derived
`projects/<slug>` under it, and searched the ambient root when that missed. Reference
resolution handed it a *project* directory as the *config root*, so the derivation
always missed. With a same-ID file in the ambient root, a Claude fork seeded from
that decoy instead of the recorded conversation. The green test suite did not catch
it; an installed-wheel probe with a decoy did. The fix was structural: one source
key, exact file, typed refusal.

## Reading transcripts

A tracked Claude chat reads `<recorded store>/<id>.jsonl` regardless of the caller's
cwd, with the same first-line `sessionId` validation as preparation. A tracked record
with no recorded store is `unbound`; its `claude_config_dir` hint is not used to
rebuild a path. Only explicitly untracked lookups use a config-root hint, and it
expands only to `<hint>/projects/<slug>`, never to ambient or default roots. See
[session operations](../codebase/session-operations.md#transcript-source-resolution).

## TUI trampoline

Entering `/tui fullscreen` in Claude's TUI starts a transient trampoline session,
and the conversation continues under a different session ID. The entry chat's
assigned ID then has little or no transcript. The real conversation is under the
successor.

**Detection** (`ClaudeAdapter.observe_after_exit`, after the attempt), file-based only:

1. If the recorded ID has a transcript, there is nothing to report.
2. Otherwise, find the recorded ID with `display: "/tui fullscreen"` in Claude's own
   `<config-root>/history.jsonl` (Claude's prompt log, not Meridian's runner history).
3. Take the next same-project prompt with a different session ID.
4. Accept it only if the successor's transcript starts with that prompt.

**What the successor means: a diagnostic, nothing more.** It is returned as
`PostExit.trampoline_successor_id` and persisted in the spawn row's `run_boundary`. It never rebinds the entry chat and never becomes an exit chat, so a Claude run's
exit stays `unresolved`. `spawn show pN` prints `entry cA (<assigned>) → exit
unresolved`, and `session log pN` shows the entry chat as an entry-based view. After
a fullscreen switch that transcript may be nearly empty. `session log cA` fails
`missing` rather than following the successor, because entry chats never move.

For one integration merge the successor was the run's exit key. The whole-change
review reproduced why that was wrong. Steps 2–4 check that B's prompt starts B's own
transcript, which ties B to B, not to A's process. An unrelated Claude chat started in
the same cwd within the window passes the same checks, even when A exited with code 1
and never made a successor. That run recorded a verified exit for someone
else's conversation. The exit-key derivation and the finalizer parameter were
deleted. A verified Claude exit needs evidence correlated to the launched child, the
way Pi's boundary record carries the launch nonce and child PID.

**Why file-based.** A `claude --resume <id> --print` probe was tested and rejected. It
performs model work that hits budget limits and is not deterministic.

**Code:** `harness/claude.py`, `harness/claude_sessions.py`,
`harness/claude_preflight.py`. Tests:
`tests/integration/launch/test_launch_process_claude_session.py` (successor persisted
as a diagnostic; an unrelated concurrent candidate leaves exit unresolved;
contradictory `system/init` fails `entry_mismatch`),
`tests/integration/harness/test_adapter_ownership.py`.

## Related

- [Native session identity decision](../decisions/native-session-identity.md)
- [Native session binding](native-session-binding.md)
- [Pi native sessions](pi-native-sessions.md): the analogous shared-store hazard for Pi
- [Codebase: harness adapters](../codebase/harness-adapters.md): Claude PTY capture, system-prompt channel
- [Launch harness compatibility](../decisions/launch-harness-compatibility.md): the original trampoline decision

**Provenance:** `work:native-harness-session-identity`; Lane C `spawn:p7041`, review
`spawn:p7056`, recheck `spawn:p7062` (decoy reproduction), fix passes `spawn:p7058`,
`spawn:p7064`; trampoline exit wiring `spawn:p7063`, removed after whole-change review
`spawn:p7072` by fix pass `spawn:p7076` (recheck `spawn:p7077`).
