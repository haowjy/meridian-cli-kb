# Decision: Pin each chat to one native session; wrap native transcripts

**Status: settled 2026-09-24; extended through probe-fix on 2026-09-26.** The combined
implementation is PR #534 (`feat/native-session-identity` @ `08499af0`, against
`main`, not merged); #520, #526 and #531 are closed as superseded. The code and live-probe
reconciliation are captured in the phase table below.
- exact entry for Pi, Claude, Codex and OpenCode, and Pi exit mapping;
- the one-time [legacy import](legacy-native-import.md);
- Pi reopen-lineage reads;
- the foundation restructure: one binding rule, one adapter template, one runner
  pipeline.

Reads and search moving off runner history, and runner history no longer being
written, are the [native-only history](native-only-history.md) decision. See
[Phases](#phases). How the seams work:
[native session binding](../architecture/native-session-binding.md). Pi specifics:
[Pi native sessions](../architecture/pi-native-sessions.md); Claude specifics:
[Claude native sessions](../architecture/claude-native-sessions.md).

## Decision

A Meridian chat `cN` is permanently associated with one **native key**:
`(harness, native_store, native_session_id)`. The store is part of the key because
the same ID in a different directory or database is a different conversation. A chat
is not a run, a retry, a transcript snapshot, or a pointer to "whatever the harness
has open now."

- **`--continue cN` resumes exactly that key or fails before creating chat or spawn
  state.** This also applies to unsupported resume/fork and harness-mismatch
  continuations; neither silently resumes in place nor starts a fresh session. The
  primary and spawn paths share the refusal message. Failure is typed:
  `unbound` (the chat never acquired a key), `missing` (the key's transcript is gone
  or not yet persisted), or `ambiguous_native_file` (more than one file claims the
  key). There is no fallback to another candidate ID, primary metadata, an adapter
  scan, an ambient harness root, or runner history. A tracked reference without a
  recorded harness refuses instead of guessing one. Reads follow the same rule: a
  tracked chat or spawn needs its complete recorded key `(harness, store, id)`, and
  a record missing its store is `unbound` even when a legacy `claude_config_dir`
  hint would locate a same-ID file. Hints serve only explicitly untracked lookups.
  Reads and launches never repair an incomplete key. The only repair is the one-time
  [legacy import](legacy-native-import.md), which runs once per
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
  copy, and it creates a second candidate authority. Reads stopped using it in PR 2 and
  writes stopped in PR 3. Old runner-history files are not decoded at all (the user
  chose option C). SQLite stays a disposable search and preview projection, never
  binding or transcript authority. See [native-only history](native-only-history.md).
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

## One binding rule, one drift rule

**Binding.** A chat's key only gains fields; it never changes them. One pure rule
decides every write and every replay: `bind(prior, attempted)` returns
- `Bound` when empty fields fill;
- `Same` when nothing is new;
- `Conflict` when a non-empty field differs.

A conflict keeps the prior key and is logged once, by the writer. Bind sources are
`assigned` (the pre-exec target), `observed` (an owned signal) and `legacy_import`.
All three obey the same rule; none can overwrite another.

**Drift.** An observed session ID fails a run only when it contradicts something
Meridian fixed before exec:
- if Meridian assigned an ID, the attempt's first owned signal must equal it;
- a fork whose ID the harness assigns must not come back with its source's ID
  (`fork_reused_source`).

Every other observed ID is diagnostic. It may complete a key that has no ID yet, but
it never fails the run and never changes the key. The connection's *current* ID is
always diagnostic, because transports overwrite it on legitimate switches.

**Why one rule.** Before the restructure, three runner copies had drifted apart. A
post-exit contradiction failed the process runner, only warned in the streaming
runner, and was never checked in `streaming serve`. Folding them into one pipeline
named seven behavior changes, each with a red-first test. Two are material:
- a Claude streaming spawn whose first ID contradicts its `--session-id` now fails
  `entry_mismatch`; before, it only warned;
- a Claude or OpenCode fork that reuses its source ID now fails in every runner.
  Before, attach and post-exit bound the fork chat to the source's key, so two
  chats held one key.

**Keeping retries working.** Codex and OpenCode create retries still succeed. The
first-signal check compares only against pre-exec facts, so attempt 2's new thread
ID is diagnostic and the entry keeps attempt 1's key.

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

**The run's record.** The spawn row keeps its entry chat as `chat_id`, which is
immutable. It records one `run_boundary` outcome:

| Status | Meaning | `exit_chat_id` |
|---|---|---|
| `verified` | An owned exit key was observed, and its native file exists | set (the owning or new chat) |
| `unresolved` | No owned exit signal (Claude, Codex, OpenCode today), or Pi ended without a readable quit | none |
| `mismatch` | The run failed `entry_mismatch` | none |

"Continue after this run" has one rule, `continue_chat_id`: a terminal run's
verified exit chat, otherwise the entry chat. The primary exit hint,
`--continue pN`, `--fork pN` and `session log pN` all use it. An exit chat is created
only when the exact native resolver finds the file. A never-saved `/new` in Pi
therefore gets no dead chat.

**Why the Claude successor is not an exit.** For one merge it was: when no adapter
boundary existed, the runners passed the successor to the finalizer as an exit key.
The whole-change review showed what the correlation actually checks. It scans
Claude's shared `history.jsonl` for the first new same-project session after A's
`/tui fullscreen` and confirms that B's prompt starts B's own transcript. That
correlates B with B, not B with A's process. In the installed wheel, A requested
fullscreen and then exited with code 1 without any successor, while an unrelated B
started in the same cwd within the window. The adapter returned B, the run recorded
a verified exit, and `session log pN` would show another person's
conversation. Exit evidence must be launch-owned (correlated to the child Meridian
started, like Pi's nonce- and PID-checked record). The `exit_key` plumbing was
deleted rather than guarded, so there is one exit source.

## Legacy chats are imported once

The user chose a bounded import for chats created before native keys were recorded.
The policy, constraints, and measured result are in the [one-time legacy import
decision](legacy-native-import.md); its implementation is documented in the
[architecture page](../architecture/legacy-native-import.md). Imported keys still
obey this page's immutable binding rule, and runner-history bytes never become proof.

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
- **Repair at read or continue time.** Reads and launches never repair incomplete
  keys. The one-time importer is followed only by an explicit late-binding repair for
  marker-listed `no_session_id` chats, run from doctor and primary-launch background
  repair; it validates exact identity and does not guess. Repairing on each read would
  make a chat's identity depend on when it was first read.
- **Fail-closed UUID minting on unreadable siblings.** A single torn journal in the
  shared primary store would block every fresh launch. Minting skips unreadable
  headers with a warning. Source resolution and post-exit verification stay
  fail-closed.
- **Failing a run on any observed ID that differs from the key.** Codex and
  OpenCode create retries legitimately see a new thread ID on attempt 2, and a TUI
  may switch sessions. Only a first signal that contradicts a pre-exec fact is fatal.
- **Logging conflicts during replay.** The fold re-reported every historical
  multi-ID chat on every command: 153 warnings per command on real data. Writers
  log once, when a conflict is attempted.
- **Letting the session store raise on conflict.** Whether a conflict is fatal
  depends on whether the signal was the attempt's first, which only the run knows.
  `SessionAttempt.bind` records and mirrors, and `NativeRun` decides.
- **A runner-side `conclude_native_identity` on the adapter.** The adapter only
  observes and returns a pure `PostExit`. The runner pipeline persists and decides,
  so there is one "conclude" and a side-effect-free adapter.

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
- **Only Pi reports an exit identity.** Every Claude, Codex and OpenCode run records
  `run_boundary: unresolved`. This is the design, not a defect: none of them has a
  launch-owned exit signal. A read-only tally of about 110 recent real rows found 0
  verified. What users see: `spawn show pN` prints `exit unresolved`, and
  `--continue pN`, `--fork pN` and `session log pN` use the entry chat. The runner-history
  prune rule skips these rows as `exit_unresolved`
  ([prune rule](native-only-history.md#the-prune-rule)).
- Claude runs never report a verified exit, including after `/tui fullscreen`.
  `session log pN` shows the entry chat with an entry-based label, and the entry
  chat's transcript may be nearly empty when the conversation moved to a successor.
  `/clear` creates a new native transcript that remains untracked (#533). Closing these
  gaps needs launch-correlated Claude evidence, not a better scan.
- Model/provider selection on reopen is a separate concern and never changes identity.

## Phases

| Phase | Scope | State |
|---|---|---|
| Identity foundation | Immutable binding, exact entry, Pi exit observation, one-time import, foundation restructure | Combined PR #534 @ `08499af0`; live Claude, Codex, OpenCode and Pi probes passed supported paths; Cursor was not probed (user does not use it) |
| Native-only history | Native reads, native-keyed rebuildable search, run facts off the stream; stop runner-history writes and once-only dogfood-row migration | Combined PR #534; probe fixes include exact OpenCode 1.x report fallback, chat-named `pN` views, text search coverage, strict `--file` admission, and explicit archive capture |
| Browse and late legacy binding | Per-row browse degradation; late exact binding only for rows missed by the one-time import | Combined PR #534; repairs run in `meridian doctor` and primary-launch background repairs, never on each command |
| Continuation safety | Unsupported fork/resume and cross-harness continuation refuse before row creation | Combined PR #534; no in-place or fresh-start fallback |
| Archive | Explicit apply captures the bound native snapshot before selection; ZIP omits retired runner streams when the snapshot exists | Combined PR #534; targeted Claude, Codex, OpenCode and Pi re-probe passed |

Verification included isolated installed-build probes of supported workflows across
Claude, Codex, OpenCode and Pi, targeted archive/read/search re-probes, and focused/full
test gates recorded in the work item. Cursor was not probed (the user does not use it).
The combined re-probe still logged a failed cross-harness spawn-continuation message
and an omitted unbound OpenCode browse row. Later continuation refusal was covered by
operations-seam regressions; late binding and browse behavior were exercised on a
copied runtime. Do not describe every formerly failing live scenario as rerun green.
The first probe failures also included wrong expectations: Claude `output.jsonl` is
process-runner-only, and TUI probing requires a separate Enter after staging text in
tmux. Claude `/clear` remains untracked, and Claude/Codex/OpenCode exit identity
remains unresolved because only Pi reports a launch-owned exit identity.

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

Foundation restructure and install: `design/pr1-foundation-restructure.md`; thermo
reviews `spawn:p7083`–`spawn:p7085`; phases `spawn:p7100`, `spawn:p7102`,
`spawn:p7114`, `spawn:p7104`, `spawn:p7116`; recheck `spawn:p7120`
(`review/pr1-thermo-recheck.md`), alignment `spawn:p7121`
(`review/pr1-alignment.md`), fix pass `spawn:p7126`
(`evidence/pr1-p5-fixes-report.md`); install readiness
`evidence/pr1-install-readiness-report.md`; conversation `chat:c6945`.
