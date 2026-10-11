# Architecture: Claude Native Sessions

Claude Code keeps every conversation as one JSONL transcript in a directory that all
Claude processes with the same config root share. This page explains how Meridian
keeps chats pinned to the right transcript in that shared store. It covers the
store layout, how a resume or fork source reaches the child, how a launch-owned
boundary receipt identifies a final session, and how the TUI trampoline is
handled. The cross-harness rule is the
[native session identity decision](../decisions/native-session-identity.md). The
shared seams are in [native session binding](native-session-binding.md). The Claude
identity work described here shipped in v0.7.0 (PR #534, merge `3dae2434`).

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

## One-attempt identity

A Claude create launch binds its `--session-id` to the chat before exec and invokes
Claude once. An exact typed terminal result, including quota exhaustion, remains the
causal final result; Meridian does not reuse the assigned identity in a second launch.
Cleanup still owns the complete process scope, and post-exit observation still checks
the exact native path for the one turn. See [native session
binding](native-session-binding.md#one-attempt-causality).

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
`PostExit.trampoline_successor_id` and persisted in the spawn row's `run_boundary`. It never rebinds the entry chat and never becomes an exit chat. By itself it does not verify an exit; a run without a valid boundary receipt remains
`unresolved`. A possible successor produces an explicit warning naming
the candidate and the recorded entry chat; it is not a verified continuation target.
Without a successor, `spawn show pN` says `Exit-session identity: unverified`, while
primary completion omits that absence-only diagnostic. The process result and saved
continue command remain visible. This presentation rule is settled in
[PR #559](https://github.com/haowjy/meridian-cli/pull/559); it does not add exit proof.
For an unresolved run, `session log pN` still shows the entry chat as an entry-based
view. After a fullscreen switch that transcript may be nearly empty. `session log cA` fails
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

## Launch-correlated session-boundary receipt

Meridian loads the separate `meridian-session-boundary` classic command-hook
plugin for both primary and spawned Claude launches. The optional idle plugin
remains interactive-only; it is not an exit-identity source. Before exec, the
Claude adapter supplies a fresh launch nonce, an absolute Python interpreter,
and a spawn-owned `claude-session-boundary.json` path through private
environment variables.

`SessionStart` records the initial native session key for `startup`, `resume`, or
`fork`, and every later start/end event replaces the current key
without changing the initial one. A `clear` or `resume` end is a switch, never
final evidence; only `prompt_input_exit`, `logout`, or `other` can identify a
final active chat. The recorder validates the session ID and exact
`<session-id>.jsonl` path, writes a bounded mode-0600 receipt atomically, and
does not recreate a deleted spawn directory. Its parent PID must remain the
launched Claude child, so a nested Claude inheriting the environment cannot
replace the receipt.

After teardown, `ClaudeAdapter.observe_after_exit()` accepts the receipt only
when its version, nonce, PID, and initial key are valid. The native-run
finalizer compares that initial key with the immutable planned entry (a fork
without a preassigned ID may complete its entry through this owned observation,
but may not reuse the fork source), then validates the exact final transcript
and its native header before allocating an exit chat. Claude performs this exact
final-transcript check even when the exit key already has a chat alias, because
the core allocator may reuse such an alias without invoking its existence
callback. The original entry chat binding never moves. Missing, stale, corrupt,
overlarge, late, or hard-killed
receipts remain unresolved; an initial mismatch is a mismatch. No transcript
directory scan, timestamp, or trampoline guess substitutes for the receipt.

Built-wheel Meridian qualification now covers the real primary `/clear` → `/exit`
path (verified new exit chat and exact `--continue`), a real cheap Haiku spawn
(receipt PID, process PID and exact transcript agree), and the 0.9.3 native fork
`c12` → `/clear` → `/exit` path (verified exit chat, immutable fork entry, and
unchanged source transcript). Claude 2.1.296's actual fork startup uses
`SessionStart.source=fork`; commit `b74dc0b3` recognizes that source alongside
`startup` and `resume`. A no-turn launch with no native transcript remains
unresolved, and the earlier p10 result claiming it was verified was an allocator
callback bypass fixed by validating every final transcript in the Claude adapter.
The broad full gate was still running at this capture; these results do not claim
universal hard-kill, crash, logout, or empty-input coverage.

**Code:** `harness/claude.py`, `harness/claude_boundary.py`,
`harness/claude_sessions.py`, `harness/claude_preflight.py`,
`launch/native_run.py`, and `claude_runtime/meridian-session-boundary/`. Tests:
`tests/integration/launch/test_launch_process_claude_session.py` (successor persisted
as a diagnostic; an unrelated concurrent candidate leaves exit unresolved;
contradictory `system/init` fails `entry_mismatch`),
`tests/integration/harness/test_adapter_ownership.py`, and the focused boundary
and projection tests.

## Related

- [Native session identity decision](../decisions/native-session-identity.md)
- [Native session binding](native-session-binding.md)
- [Pi native sessions](pi-native-sessions.md): the analogous shared-store hazard for Pi
- [Codebase: harness adapters](../codebase/harness-adapters.md): Claude PTY capture, system-prompt channel
- [Launch harness compatibility](../decisions/launch-harness-compatibility.md): the original trampoline decision

**Provenance:** `work:native-harness-session-identity`; Lane C `spawn:p7041`, review
`spawn:p7056`, recheck `spawn:p7062` (decoy reproduction), fix passes `spawn:p7058`,
`spawn:p7064`; trampoline exit wiring `spawn:p7063`, removed after whole-change review
`spawn:p7072` by fix pass `spawn:p7076` (recheck `spawn:p7077`). Boundary design and
implementation: `work:exit-identity-reporting`, commit `3aa18aae`, review
`spawn:p7571`; classic runtime probe: `claude-boundary-live-probe.md`; final
built-wheel evidence `claude-final-live-probe.md` and
`claude-fork-clear-final-proof.md`; alias-validation fix `3630a1f0b` and fork
source fix `b74dc0b3`. The broad full gate was still running at this capture.
