# Decision: Import legacy native session keys once

**Status: settled 2026-09-25; old-Pi recovery settled 2026-09-26.** The one-time
migration decision, its constraints, and the [old Pi chat recovery](#old-pi-chats-that-067-never-bound-content-proven-recovery-manual-repair)
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

## Old Pi chats that 0.6.7 never bound: content-proven recovery, manual repair

**Settled 2026-09-26.** The user asked to fix old Pi chats "or provide a way to".

**The gap.** Real 0.6.7 never recorded the ID of a headless Pi spawn, and some
primaries also ended with `harness_session_id: null` (`discovery_failed`). The import
marks them `no_session_id`: 770 Pi chats across three runtimes (576 in meridian-cli,
460 spawned and 116 primary). The native files exist. 0.6.7 ran headless Pi with
`--session-dir <pi root>/<spawn-id>/` (root `~/.meridian/meridian-pi/sessions`), but
its observer never learned the ID. Late binding cannot help, because it needs a
recorded ID.

**Why uniqueness is not proof.** The first rule measured was structural: the chat
maps to a spawn, the spawn's directory holds exactly one Pi session, the header cwd
matches, the start falls in the chat's time window, and no other chat claims the ID.
It proved 30 chats, and one of them, `c8145`, would have bound the wrong session: its
only candidate's first user message was not the chat's prompt. A single wrong bind is
permanent, because bindings are immutable. So every automatic bind needs **content
proof** as well.

**The automatic rule.** A one-shot pass binds a chat only when all of these hold:

1. The chat is a spawned chat (never a primary) that maps to a spawn through spawn
   rows, `sessions.jsonl` `spawn_id`, or archive receipts for reclaimed spawns.
2. The candidate directory is the spawn's own session dir (the recorded
   `pi_runtime_meta` `session_dir`, else `<pi root>/<spawn-id>`). The shared root is
   never a candidate dir.
3. Exactly one valid Pi session (header version 1 to 3) survives: its header cwd is
   one of the recorded `execution_cwd`, `task_cwd` or `control_root`; its start is
   within 120 s of the chat's or spawn's run window; and its ID is bound to no chat
   and claimed by no other chat's candidates.
4. Content proof: the first user message equals the retained starting prompt
   (whitespace-normalized). Only if no prompt was retained, the final assistant text
   must equal the report body. A mismatch, conflicting prompts, or nothing retained
   means no bind.

**Primaries are never automatic.** In 0.6.7 every primary wrote to the one shared
root, so same-cwd primaries compete for the same pool. In the measurement none of 109
meridian-cli primaries had a unique candidate, and `c8145` itself is a primary whose
likely candidate is the next primary launch 21 s after it stopped. A user decides.

**Result.** On copies of the three runtimes the pass bound **155 of 770** (63, 91
and 1), each by an exact prompt match, with no wrong bind found in a by-eye check of
10. `c8145` stays unbound. What is left: 419 spawned chats map to no spawn anywhere,
124 are primaries, 71 have no valid Pi file in the spawn dir, and one has neither
prompt nor report. The yield rose from 30 because 0.6.7 ran Pi in the control root,
so header cwds equal `control_root`, not the worktree `execution_cwd` the first
measurement checked.

**Manual path.** `meridian session repair cN` (or its spawn `pN`) is read-only
without `--native`: for an unbound chat it lists candidate native files with
evidence (path, session ID, start, cwd, first-message excerpt, and whether cwd, time
window, prompt and other bindings match) and prints the exact bind command. `--native
PATH` validates the file for the chat's harness and binds it. `--force` is needed for
a cwd mismatch or a start outside the time window. Repair always refuses a chat that
is already bound, a session bound to another chat, and a file that is not a valid
native session for the chat's harness; `--force` never overrides those. Repair takes
chat and spawn refs only, not raw harness IDs. Bindings stay immutable.

The old `session repair` wrote only a bare `harness_session_id` (an "observed" ID with
no store), which is an incomplete key under the exact-key rule. It was deleted, not
adapted.

**Bind sources.** The automatic pass binds with `legacy_pi_recovery`, and the manual
path with `user_repair`. Both go through the same lock-scoped `bind()` as every other
source. Mechanism: [legacy native import](../architecture/legacy-native-import.md#legacy-pi-recovery).

**Rejected:**
- **Uniqueness alone** (the structural rule): it bound `c8145` to the wrong session.
- **Automatic primary binding from the shared root**: no primary had a unique
  candidate, and time-adjacent launches look alike.
- **Guessing a spawn dir for unmapped chats**: `pNNN` IDs are project-local while the
  Pi root is user-global, so the same `<spawn-id>/` can belong to another project.

**Revisit if** a later measurement finds a wrong bind under content proof, or Pi
session directories become globally unique per spawn.

**Provenance:** `work:native-harness-session-identity`, `decision.md` entry "Old Pi
chat recovery + upgrade guide" (2026-09-26); measurement `spawn:p7228`
(`evidence/measure-pi-legacy-recovery.md`); implementation `spawn:p7229` (commits
`1ab75dee`, `0245ae64`); earlier finding `spawn:p7213`
(`evidence/probe3-upgrade-rerun.md`).
