# Decision: Pin each chat to one native session; wrap native transcripts

**Status: settled 2026-09-24; extended 2026-09-25** (one recorded source key,
qualified-event identity for every harness, observed exit identity, Claude exit
unresolved). Exact entry for all four tracked harnesses and Pi exit mapping are
implemented on `fix/native-session-wrapper` (draft PR #520, head `95db4d03`), which
passed a whole-change review on recheck. They are not merged to `main`. The one-time
[legacy import](#chats-from-before-the-key-existed-import-once) is on slice branch
`slice/pr1-legacy-import` (implementation `8e8fe485`, docs head `fc234735`), which will merge into PR #520. Native readers
and runner-history removal have not started (see [Phases](#phases)). How the seams work:
[native session binding](../architecture/native-session-binding.md). Pi specifics:
[Pi native sessions](../architecture/pi-native-sessions.md); Claude specifics:
[Claude native sessions](../architecture/claude-native-sessions.md).

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
  recorded harness refuses instead of guessing one. Reads follow the same rule: a
  tracked chat or spawn needs its complete recorded key `(harness, store, id)`, and
  a record missing its store is `unbound` even when a legacy `claude_config_dir`
  hint would locate a same-ID file. Hints serve only explicitly untracked lookups.
  Reads and launches never repair an incomplete key. The only repair is the one-time
  [legacy import](#chats-from-before-the-key-existed-import-once), which runs once per
  runtime root and then stops.
- **Two typed failures, one route.** A contradiction (observed identity ≠ the key) is
  `NativeEntryMismatch` with stable code `entry_mismatch` and structured
  expected/observed evidence. Unavailability is
  `NativeSessionUnavailable(ref, unbound | missing | ambiguous_native_file)`. Both
  reach terminal classification through the same path (runner finalization, or
  `ops/spawn/failure_policy` for refusals before the runner), and neither permits exit
  allocation or invocation attribution. Before the whole-change fix pass, Pi's
  verifier returned formatted strings, managed-primary attach raised `RuntimeError`,
  and a changed-store assignment raised `ValueError`, so the same contradiction
  surfaced as three different failures.
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
| Resume | Verifies the exact locator: one file in the recorded store whose native header carries the key's ID | Harness opens that path/ID as-is |
| Fork | Verifies the source the same way; assigns the new target ID where the harness accepts one | Harness writes a new session that records its parent |

**A locator is verified by its content, not its filename.** Pi's `session` header
`id`, Claude's first-line `sessionId`, and Codex's `session_meta.payload.id` must
equal the key. An empty, torn, or malformed header is *unavailable* (`missing`); a
readable header with another ID is a *contradiction* (`entry_mismatch`). The first
Claude/Codex implementation checked only that the named file existed (Claude) or
that one rollout filename matched (Codex). The whole-change review reproduced a Claude
file named for A that held B's events being read and resumed as A, and tests that
blessed `"{}\n"` as a resumable Codex journal.

The binding happens under the sessions lock and is bind-once. The first owned
identity signal the harness emits (Pi `session_start`, Codex/OpenCode API response,
Claude `system/init` or connection ID) confirms it. If that signal contradicts the
target, the attempt fails as `entry_mismatch`, in the primary runner as well as the
streaming runner. The check runs before the run completes and before invocation
attribution. No binding is created for the unexpected key, and no chat's key changes.
Identity switches after that are ordinary switches, not entry evidence. (The primary
Claude runner originally extracted the owned identity only after completing the spawn
and logged the contradiction; an `sh` shim emitting B while argv assigned A exited 0
as `succeeded`. Each lane's own review missed it because each lane owned one runner.)

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
  unreadable identity leaves exit `unresolved`. The record is read only after the
  child has exited and its teardown has been joined.
- **Claude** exit is always `unresolved` today. The TUI trampoline successor
  (`/tui fullscreen`) is recorded as a diagnostic `trampoline_successor_id` and
  becomes neither an entry rebind nor an exit chat.
- Pi's owned boundary is the only input to the one finalizer and one allocator. A
  verified exit key maps to the chat that already owns that exact key (stopped chats
  included), else a new chat. The lookup and creation run under the sessions lock.
- If the boundary's initial identity contradicts the immutable entry key, the run
  fails with the same typed `entry_mismatch` as a startup contradiction. No exit chat
  is created, and an attempt that already failed with a typed identity error skips
  boundary observation entirely.

Exit is presentation and ownership for the *next* conversation. It never changes
what `--continue cN` means for the entry chat.

**Why the Claude successor is not an exit.** For one merge it was: when no adapter
boundary existed, the runners passed the successor to the finalizer as an exit key.
The whole-change review showed what the correlation actually checks. It scans
Claude's shared `history.jsonl` for the first new same-project session after A's
`/tui fullscreen` and confirms that B's prompt starts B's own transcript. That
correlates B with B, not B with A's process. In the installed wheel, A requested
fullscreen and then exited with code 1 without any successor, while an unrelated B
started in the same cwd within the window. The adapter returned B, the run recorded
`exit_identity=verified`, and `session log pN` would show another person's
conversation. Exit evidence must be launch-owned (correlated to the child Meridian
started, like Pi's nonce- and PID-checked record). The `exit_key` plumbing was
deleted rather than guarded, so there is one exit source.

## Chats from before the key existed: import once

**User decision, 2026-09-25: "Auto-import once".** The exact-key rule makes every chat
created before PR 1 unusable, because none of them recorded `native_store`. On the
user's real project (about 7,000 chats; codex 4,758, claude 750, pi 683, opencode 481,
cursor 320), every `session log cN` and `--continue cN` refused as `unbound`, although
0.6.7 handled them. The design had said that old data is never upgraded into proof. The
user chose a bounded upgrade instead. The rule that runner-history bytes never become
proof still holds.

The import's rules:

- **Once per runtime root.** The first command that resolves the runtime root runs
  the import. An atomic marker file then records the outcome, including every
  chat left unbound and why. Nothing is inferred at runtime after that.
- **The ID comes from the chat's own records.** Sources are the journal's recorded
  session IDs (including historical multi-ID arrays), then the chat's own spawn rows. More than one distinct
  ID is `ambiguous_id`, and the chat is skipped. No ID at all is `no_session_id`.
- **Only the harness's own place for that ID is checked.** Candidate stores come
  from recorded facts: the recorded cwds and config dir, or the default home when
  nothing was recorded. Today's ambient environment is ignored. The import binds
  only when exactly one candidate store holds a file or row whose native header
  carries the ID. It uses the same exact validators as live reads. Zero matches is
  `missing`. More than one match is `ambiguous`. It never picks the newest match
  and never searches other projects.
- **One bind path.** Accepted keys go through the same lock-scoped bind as
  launches, with `source: legacy_import`. A chat that already has a complete key is
  never touched, and the normal conflict rules apply.
- **Unknown identity stays unsupported.** Cursor, which has no native identity in
  PR 1, and restored historical records are counted `unsupported` and are not
  changed.
- **Damaged sources defer the import.** If the import meets a quarantined spawn row,
  an I/O error, a torn or changing OpenCode snapshot, or a failed strict row query,
  it writes no marker and prints one warning, and the command continues. A failed
  source must not be recorded as a permanent `missing`, and the import must not
  block the CLI.

**Result on real state** (read-only report at `8e8fe485`): of 7,002 chats, 2,122 can
be imported and 4,880 stay unresolved. A copy-based import found log c6988 and
the dry-run continues of c6945 and c6992. The real `sessions.jsonl` was not written.

**Pi scope is a brief constraint, not a harness limit.** Pi candidates are limited to
Meridian's per-spawn Pi session dirs for the chat's own spawns, as the implementation
brief required. PR 1 records the unscoped Pi session root for interactive primaries.
The real root holds 32 top-level Pi session files; most belong to no recorded chat.
A read-only comparison against the IDs on this project's own chat and spawn rows
matched 7 of those files to 9 chats (c6386, c6424, c6435, c6463, c6518, c6689,
c6907, c6936, c6959; one file is shared by c6518, c6689, and c6907). Those 9 chats
stay `missing` under the current scope. The comparison shows where a widened scope
would find candidates. It is not a bind result: no import has run under a wider
rule. Whether to widen the candidate
set is an open decision for the work-item lead. Treat these chats as out of scope,
not as unbindable. Evidence: `evidence/pr1-legacy-import/pi-root-only-matches.json`
in the work item.

## Rejected alternatives

- **Observed-entry-only rule with a Pi RPC primary.** Rejected for the reasons above.
- **Repointing a chat on switch**, or treating every switch as fatal. Repointing
  corrupts `--continue`. A fatal switch breaks normal TUI use.
- **Mirrors as authority.** Primary metadata, spawn-row IDs, and multi-ID session
  arrays could each "recover" an identity. The chat binding is the only authority;
  mirrors copy the accepted ID.
- **Harness-specific source fields on the launch request.** Rejected after they
  produced the two defects above. Harness mechanics belong in the adapter.
- **Adopting the Claude trampoline successor as the chat's key or the run's exit.**
  A same-project prompt match is inference, not an owned signal, and it cannot tell
  A's successor from an unrelated concurrent chat. It stays a diagnostic.
- **A separate file-pin registry or conflict journal.** Resume re-resolves the exact
  file inside the recorded store and verifies its header. The store is already in the
  key.
- **Leaving legacy chats unbound** (the design's original stance). Rejected by the
  user: every existing chat would lose log and continue on upgrade.
- **Repair at read or continue time.** The chosen option says explicitly that
  nothing is guessed at runtime after the import. Repairing on read would make a
  chat's identity depend on when it was first read, and it is the kind of runtime
  guessing this decision rules out.
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
- Claude runs never report a verified exit, including after `/tui fullscreen`.
  `session log pN` shows the entry chat with an entry-based label, and the entry
  chat's transcript may be nearly empty when the conversation moved to a successor.
  Closing this needs launch-correlated Claude evidence, not a better scan.
- Model/provider selection on reopen is a separate concern and never changes identity.

## Phases

| Phase | Scope | State |
|---|---|---|
| 1 | Immutable binding, pre-exec plan seam, exact-only resolution; exact identity for Pi, Claude, Codex, OpenCode; one source key; header-validated locators; typed refusals | Implemented on draft PR #520; whole-change review PASS on recheck |
| 1b | One-time exact legacy import of native keys (user decision "Auto-import once") | On slice branch `slice/pr1-legacy-import` (`8e8fe485`), reviewed and rechecked; final gate running; merges into PR #520 |
| 2 | Pi exit observation (session-boundary extension), B→own cN, Claude exit unresolved; real-Pi 0.87.1 qualification | Implemented on draft PR #520; real-Pi create and missing-source refusal qualified; Meridian-managed continue/fork/switch not qualified |
| 3 (PR "E") | Native readers: `session log`/context/search on the exact key; Pi reopen-lineage view; rebuildable search; OpenCode report reads the recorded DB | Not started |
| 4 (PR "F") | Remove runner-history (`history.jsonl`) writers/readers/checkpoints; measure cost | Not started |

Verification standard: POSIX `sh` harness shims at the real runner seams, CLI probes
against an isolated installed wheel with decoys and unrelated concurrent sessions,
and the built Pi extension bundle run in Node and read by the production Python
reader. Real Pi 0.87.1 ran with zero model turns for lifecycle probes and once with
one authorized model turn under Meridian (isolated store and home, `--offline`,
`--no-tools`; 1,344 input / 2 output tokens, $0.0004). That run verified create (the
native header UUID equals the assigned c1; `session log c1` and `--raw` print exactly
the turn) and the typed pre-launch refusal after deleting the source file. It also
exposed the teardown-ordering defect described in
[native session binding](../architecture/native-session-binding.md#runner-order).
Meridian-managed continue, fork, and switch → exit chat against real Pi need a second
model turn or a driven TUI and have not been run. Real Claude, Codex, and OpenCode
services have not been run against this change.

Out of scope here: duplicate Pi completion-notification turns (GitHub #517).

**Provenance:** `work:native-harness-session-identity` (`decision.md`,
`DIVERGENCE/exact-locator-entry.md`, `design/native-history-only.md`); design review
`spawn:p7037`; lanes B `spawn:p7038`, A `spawn:p7040`, C `spawn:p7041`, D
`spawn:p7054`; reviews `spawn:p7039`, `spawn:p7044`, `spawn:p7045`, `spawn:p7056`,
recheck `spawn:p7062`, `spawn:p7059`; fix passes `spawn:p7058`, `spawn:p7064`,
`spawn:p7061`, `spawn:p7067`; merges `spawn:p7057`, `spawn:p7063`, `spawn:p7070`;
whole-change review `spawn:p7072` and recheck `spawn:p7077`
(`review/integrated-pr1.md`); whole-change fix pass `spawn:p7076`
(`evidence/integrated-fix-report.md`); real-Pi probes `spawn:p7065` (zero-turn),
`spawn:p7073` (stopped on credentials), `spawn:p7075` (one turn,
`evidence/lane-q3-report.md`); incident `spawn:p6615`. Legacy import: user decision
in `decision.md` ("USER DECISION — legacy chats — auto-import once"), brief
`prompts/pr1-legacy-import.md`, commits `96e146d0`, `ce8b6df5`, `8e8fe485`; review
`spawn:p7091` (`evidence/pr1-legacy-review.md`), recheck `spawn:p7093`
(`evidence/pr1-legacy-recheck.md`), report `evidence/pr1-legacy-import-report.md`,
Pi root comparison `evidence/pr1-legacy-import/pi-root-only-matches.json`.
