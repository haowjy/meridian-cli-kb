# Decision: Import legacy native session keys once

**Status: settled 2026-09-25.** The one-time migration decision and its constraints
are here; the implementation sequence, locks, candidate-store checks, recovery,
and cost are in [legacy native import](../architecture/legacy-native-import.md).
The broader identity invariant is [native session identity](native-session-identity.md).


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
- **Damaged sources defer the import.** A quarantined spawn row, an I/O error, or a
  failed strict SQLite query writes no marker. One warning is printed, and the
  command continues. A 15-minute backoff note keeps the commands in between quiet. A
  failed source must not be recorded as a permanent `missing`, and the import must
  not block the CLI.

**Result on real state.** Measured on a copy of the meridian-cli root at install
time: of 7,004 chats, 2,133 imported, in 3.75 s. The rest stay unbound, mostly
because their native files are gone:
- Claude deletes transcripts after 30 days by default;
- Codex rollouts from before 2026-06-23 are gone;
- old Pi chats never recorded an ID;
- Cursor is unsupported.

The candidate set includes Meridian's unscoped interactive Pi session root, where PR 1
records interactive primaries. That root holds 9 chats whose files a per-spawn-only
scope missed; all 9 imported. Meridian-flow imported 6,074 chats. For the chats left
unbound, runner `history.jsonl` was the only other copy. Under
[option C](native-only-history.md#old-runner-history-option-c-drop) it is not read.

## Open: old Pi chats that 0.6.7 never bound (decision pending)

**Finding (round-3 probe, 2026-09-26).** Real 0.6.7 never recorded the ID of a
headless Pi spawn, and some primaries also ended with `harness_session_id: null`
(`discovery_failed`). The import marks them `no_session_id`: 576 Pi chats in the
meridian-cli runtime (460 spawned, 116 primary) and 193 in another project.
The native files exist. 0.6.7 ran headless Pi with `--session-dir <pi
root>/<spawn-id>/` (the root is `~/.meridian/meridian-pi/sessions`), but its observer
never learned the ID. The connection sets it only from a Pi `session` event, the
extractor needs an `output.jsonl` these spawns lack, and the file-scan fallback
scans only the root, not the per-spawn subdirectories. Late binding cannot help,
because it needs a recorded ID. Live old primaries do rebind after they exit (the
c88 case).

**Why no automatic recovery yet.** The only evidence is the directory. A chat maps
to a spawn, the spawn's directory holds exactly one file, and the header `cwd` equals
the chat's recorded `execution_cwd`. That proves only 28 chats (23 + 5). `pNNN` IDs
are project-local while the Pi root is user-global, so the same `<spawn-id>/`
directory can belong to spawns from different projects. Binding on it would guess.
No chat carries a structured ID event, so the stronger rule (a recorded ID plus
exactly one matching header) binds none.

**Current behavior.** These chats stay unbound. `session log`, search and
`--continue` refuse them. `spawn show` still shows the report, and the native file
is readable with `session log --file <path>`.

**Recommended, awaiting the user:** document this as a known limit, and file an
issue for a guarded one-shot recovery in the late-binding path. It would bind only
with the per-spawn directory, one file and a matching cwd header, and refuse on any
cross-project spawn-ID collision. Future Pi session directories would get globally
unique names. Evidence: investigation `spawn:p7213`,
`evidence/probe3-upgrade-rerun.md`.
