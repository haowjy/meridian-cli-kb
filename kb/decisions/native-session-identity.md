# Decision: Pin each chat to one native session; wrap native transcripts

**Status: settled 2026-09-24; extended 2026-09-25** (one recorded source key,
qualified-event identity for every harness, observed exit identity). Exact entry for
all four tracked harnesses and Pi/Claude exit mapping are implemented and reviewed on
integration branches for PR #520. They are not merged to `main`. Native readers and
runner-history removal have not started (see [Phases](#phases)). How the seams work:
[native session binding](../architecture/native-session-binding.md). Pi specifics:
[Pi native sessions](../architecture/pi-native-sessions.md).

## Decision

A Meridian chat `cN` is permanently associated with one **native key**:
`(harness, native_store, native_session_id)`. The store is part of the key because
the same ID in a different directory or database is a different conversation. A chat
is not a run, a retry, a transcript snapshot, or a pointer to "whatever the harness
has open now."

- **`--continue cN` resumes exactly that key or fails.** Failure is typed:
  `unbound` (the chat never acquired a key), `missing` (the key's transcript is gone
  or not yet persisted), or `ambiguous_native_file` (more than one file claims the
  key). There is no fallback to another candidate ID, primary metadata, an adapter
  scan, an ambient harness root, or runner history. A tracked reference without a
  recorded harness refuses instead of guessing one.
- **Exit may land elsewhere; the source is never repointed.** Inside a harness TUI the
  user can `/resume` or `/new`. If a run enters on c5/X and exits on Y, Y maps to its
  own existing or new chat; c5 stays X. An exit Meridian cannot observe is
  `unresolved`, and the run's view stays on its entry chat.
- **Meridian is a wrapper over harness-native transcripts.** The native journal is the
  conversation. Meridian's own `spawns/<id>/history.jsonl` runner stream is a second
  copy that creates a second candidate authority, so new runner-stream writes are to
  stop and existing files stay as labeled legacy evidence. SQLite stays a disposable
  search/preview projection, never binding or transcript authority. The runner stream
  is still written today; removal comes after native readers.
- **Not a new initiation mode.** `--from` (fresh session plus lightweight context),
  `--fork`/`--fork-fresh`, `-f`, and `spawn inject` keep their meanings. See
  [session initiation](../concepts/session-initiation.md).

## Entry evidence is a harness-guaranteed exact target

Before exec, Meridian decides the exact native target and binds it to the chat. The
target is one of:

| Operation | What Meridian does before exec | Harness guarantee relied on |
|---|---|---|
| Create | Mints the ID (Pi, Claude) or obtains it from an owned pre-input response (Codex `thread/start`, OpenCode `POST /session`) | Harness creates or reopens exactly that ID |
| Resume | Verifies the exact locator (file exists, header ID equals the key, store matches) | Harness opens that path/ID as-is |
| Fork | Verifies the source; assigns the new target ID where the harness accepts one | Harness writes a new session that records its parent |

The binding happens under the sessions lock and is bind-once. The first owned
identity signal the harness emits (Pi `session_start`, Codex/OpenCode API response,
Claude hook or connection ID) confirms it. If that signal contradicts the target, the
attempt fails as `entry_mismatch`. No binding is created for the unexpected key, and
no chat's key changes. Identity switches after that are ordinary switches, not entry
evidence.

**Why argv counts as evidence here.** The earlier rule was "planned argv is not an entry
binding". It demanded an *observed* identity before any input reached the model. On Pi
that meant an owned pre-input gate, which exists only in `--mode rpc`. Meridian has no
RPC frontend for the interactive TUI that people actually use, so primary Pi could
never be tracked under that rule. The comparison branch that built toward it (PR #519,
~22k lines: RPC-primary route, raw-argument grammars R1/R2, model-intent fold C1–C3,
owner/coordinator layers, `transport_unqualified` refusal) never enabled a single
tracked continue. The correction is to ask what the harness itself guarantees. When
Meridian verified the target and the harness contract says it opens exactly that
target, the emitted operation is evidence. Anything the contract does not cover is
disclosed as a limit instead of being hidden behind more machinery.

Some pieces of that branch were kept: the Pi session-boundary extension (for exit
observation), the Pi reopen-default lineage projector (for native reads), and refusal
of raw passthrough flags that would override managed identity.

## One recorded source key

"Which native conversation does this resume or fork start from?" has exactly one
carrier: the source chat's recorded `(native_store, native_id)`, carried as
`SessionRequest.source_native_store` plus the requested native ID from reference
resolution through continue replay, the fork request builder, and spawn execution
to the adapter. Each adapter translates that store into its own harness mechanics in
harness code: Codex sets `CODEX_HOME` from the store, OpenCode sets `OPENCODE_DB` to
the exact recorded database, Pi sets `PI_CODING_AGENT_SESSION_DIR` and
`--session-dir`, and Claude seeds `<store>/<id>.jsonl` into the child's own project
store. `launch/` and `ops/` hold no harness-specific source fields.

**Why one key.** Before this, three request fields described the same concept
(`source_native_store`, a Claude config-root field, a Pi session-dir field). Each
carrier had its own producer and consumer, and they drifted. Two blocking defects
came straight from that:

- The fork request builder copied the Claude/Pi fields but dropped
  `source_native_store`, so Codex fork materialization ran against the ambient
  `CODEX_HOME`. Tracked forks with a recorded store could not work.
- Reference resolution put a Claude *project* directory into the *config-root*
  field. Claude preparation then re-derived `projects/<slug>` under it, missed, and
  searched the ambient root. With a same-ID decoy in the ambient root, a Claude fork
  started from the decoy instead of the recorded conversation.

Both were fixed in one pass by deleting the extra fields, not by patching each
consumer. The same pass made every source read exact. There is no ambient
model-reader branch, and a Claude source is exactly `<recorded store>/<id>.jsonl`
or a typed `missing`.

## Identity comes only from qualified events

Identity evidence is the assigned pre-exec target plus owned, qualified identity
events or API responses. Nothing else counts, for any harness:

- no choosing a journal by cwd, mtime, newest file, or ID prefix;
- no ambient harness root when a recorded store exists;
- no regex over assistant or tool text (`codex resume <uuid>` in model prose once
  bound a fabricated ID);
- no recursive search for identity-shaped keys inside nested payloads.

Incident p6615 is the reason for the first rule. A fresh Pi primary had no assigned
ID, so Meridian took the newest same-cwd journal in a shared directory, and a
concurrent same-cwd session could be selected. The wrong ID was persisted as
canonical, and a later resume loaded someone else's conversation. That was real
model-context contamination, not a display bug. The prose and nested-key cases are
the same failure through a different door: once binding is immutable, a fabricated
first observation is frozen forever.

When a harness assigns the ID only after start (Codex/OpenCode create, Claude fork),
the chat stays unbound until the first owned event, and that event binds ID *and*
store together. Binding an ID without its store left later reads to fall back to
ambient roots.

## Exit identity is observed or unresolved

The entry key never moves. What the user ended on is a separate fact:

- **Pi** reports it through Meridian's session-boundary extension. Only a final
  `session_shutdown` with reason `quit` that carries a readable identity verifies
  exit. Shutdown for a switch, a missing record, a restart after quit, or an
  unreadable identity leaves exit `unresolved`.
- **Claude**'s TUI trampoline successor (`/tui fullscreen`) is recorded as a
  diagnostic `trampoline_successor_id`. It is never an entry rebind. When the
  adapter produces no boundary, the successor becomes the run's exit key.
- Both feed one finalizer and one allocator. A verified exit key maps to the chat
  that already owns that exact key (stopped chats included), else a new chat. The
  lookup and creation run under the sessions lock.
- If the boundary's initial identity contradicts the immutable entry key, the run
  fails with the same typed `entry_mismatch` as a startup contradiction. No exit chat
  is created.

Exit is presentation and ownership for the *next* conversation. It never changes
what `--continue cN` means for the entry chat.

## Rejected alternatives

- **Observed-entry-only rule with a Pi RPC primary.** Rejected for the reasons above.
- **Repointing a chat on switch**, or treating every switch as fatal. Repointing
  corrupts `--continue`. A fatal switch breaks normal TUI use.
- **Mirrors as authority.** Primary metadata, spawn-row IDs, and multi-ID session
  arrays could each "recover" an identity. The chat binding is the only authority;
  mirrors copy the accepted ID.
- **Harness-specific source fields on the launch request.** Rejected after they
  produced the two defects above. Harness mechanics belong in the adapter.
- **Adopting the Claude trampoline successor as the chat's key.** A same-project
  prompt match is inference, not an owned signal. It may name the run's exit, never
  the entry.
- **A separate file-pin registry or conflict journal.** Resume re-resolves the exact
  file inside the recorded store and verifies its header. The store is already in the
  key.
- **Fail-closed UUID minting on unreadable siblings.** A single torn journal in the
  shared primary store would block every fresh launch. Minting skips unreadable
  headers with a warning. Source resolution and post-exit verification stay
  fail-closed.

## Accepted limits

- External deletion or replacement of a verified file between preflight and the
  harness opening it is outside the guarantee. Pi, for example, initializes a new ID at
  a missing path. Detection fails the attempt, but input may already have reached the
  model.
- Collision checks rule out an existing ID when they run. They are not an atomic
  reservation against external writers. The protection is UUID4 improbability plus
  the check.
- A Pi create that never produced an assistant message has no file. It is `pending`,
  and continuing it fails `missing`.
- A Pi run whose stdin closes while a session replacement is still settling can end
  with no readable quit identity. Its exit is `unresolved`, not guessed.
- Model/provider selection on reopen is a separate concern and never changes identity.

## Phases

| Phase | Scope | State |
|---|---|---|
| 1 | Immutable binding, pre-exec plan seam, exact-only resolution; exact identity for Pi, Claude, Codex, OpenCode; one source key; typed refusals | Implemented and reviewed on integration branches; draft PR #520 |
| 2 | Pi exit observation (session-boundary extension), Claude successor as exit key, B→own cN; isolated real-Pi 0.87.1 qualification | Implemented; zero-turn real-Pi probe done; one-turn end-to-end qualification not run yet |
| 3 | Native readers: `session log`/context/search on the exact key; Pi reopen-lineage view; rebuildable search | Not started |
| 4 | Remove runner-history writers/readers/checkpoints; measure cost | Not started |

Verification standard: POSIX `sh` harness shims at the real runner seams, CLI probes
against an isolated installed wheel, and the built Pi extension bundle run in Node
and read by the production Python reader. Real Pi 0.87.1 ran only with zero model
turns (temporary store, `--offline`, `--no-tools`, isolated home). Pi writes no
session file before an assistant message, so real create → continue → fork → switch →
exit needs one bounded model turn. That is the remaining qualification. Real
Claude, Codex, and OpenCode services have not been run against this change.

Out of scope here: duplicate Pi completion-notification turns (GitHub #517).

**Provenance:** `work:native-harness-session-identity` (`decision.md`,
`DIVERGENCE/exact-locator-entry.md`, `design/native-history-only.md`); design review
`spawn:p7037`; lanes B `spawn:p7038`, A `spawn:p7040`, C `spawn:p7041`, D
`spawn:p7054`; reviews `spawn:p7039`, `spawn:p7044`, `spawn:p7045`, `spawn:p7056`,
recheck `spawn:p7062`, `spawn:p7059`; fix passes `spawn:p7058`, `spawn:p7064`,
`spawn:p7061`, `spawn:p7067`; merges `spawn:p7057`, `spawn:p7063`; real-Pi probe
`spawn:p7065`; incident `spawn:p6615`.
