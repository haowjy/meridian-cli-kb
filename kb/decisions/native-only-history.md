# Decision: Conversation history is native-only

**Status: settled 2026-09-25; PR 2 landed on its branch.** User decisions: three
stacked PRs; old runner history option C (drop).
- **PR 2** moves reads, search and run facts off runner `history.jsonl`. It is draft
  PR #526 (`feat/native-reads` @ `3ae3fce8`, stacked on PR #520), not on `main`. Every
  slice is merged: R1, R2a, R2b, R3, A1, F1a, F1b, F2 and V1. It went through a
  thermo-nuclear review, a design-alignment review, fix lanes A–D, and a recheck that
  passed. Before/after measurement on runtime copies is done.
- **PR 3** stops writing the stream. It is not started; its scope is
  `evidence/pr3-deletion-list.md`.

How the reads work: [native transcript reads](../architecture/native-transcript-reads.md).
The identity rule this builds on: [native session identity](native-session-identity.md).

## Decision

A chat's conversation is the harness's own transcript for the chat's bound native
key. Meridian reads it through one path:

**ref → chat → the chat's `NativeKey` → the adapter's exact resolver → that
harness's reader.**

- **Implicit reads.** `session log`, export, preview/browse, `search REF` and
  corpus search never read runner `history.jsonl` implicitly, and neither do
  report, usage and failure extraction.
- **Explicit reads.** A user can point `session log --file` at a native file. A
  runner-history file given to `--file` is rejected as "not a native transcript".
- **Failure is typed.** When a chat has no key, the answer is
  `NativeSessionUnavailable(unbound)`. When the key's file is gone, the answer is
  `missing`. Meridian never falls back to runner history, a spawn directory or an
  index row.
- **SQLite only accelerates.** The metadata index and the search projection
  (below) are disposable. Neither can choose which native source a chat names, and
  deleting either one costs only time.
- **The run stream goes away.** Meridian's `spawns/<id>/history.jsonl` stops being
  written in PR 3. Live event delivery to subscribers stays in memory. The few
  facts that outlive the run are written as bounded files; they are not a stream.

### Why

The user's framing:
- The goal of the work item was to "get rid of my kind of stupid idea of having some
  kind of intermediary meridian transcript and just focus on myself just being a
  wrapper basically".
- The PR plan (2026-09-25): "2. stop relying on history.jsonl for anyhting … i don't
  think claude even relies on history.jsonl… 3. stop creating history.jsonl at all".
- Earlier the same day: switching mid-session "looks like it will break
  history.jsonl and is just extra processing that we are basically wasting".

- **It is wrong after a switch.** A managed run's stream records whatever the
  harness emitted under the run's entry chat. After a `/resume` or `/new` inside the
  TUI, one stream holds two native conversations. Every reader that falls back to
  the stream then shows or derives facts from a mixed transcript. Native reads keyed
  by the [immutable chat binding](native-session-identity.md) avoid this by
  construction.
- **It duplicates the harness.** On the user's machine, runner history was 27.1 GB
  in 4,392 files, out of 29 GB under `~/.meridian/projects`. For bound chats it
  repeats what the native files already hold. It adds no reasoning either:
  - Claude headless spawns have almost no thinking text in either copy.
  - Claude TUI native files carry thinking text; the runner copy does not.
  - Codex reasoning is encrypted in the native file and absent from the runner copy.
- **It is a second authority.** Each fallback to the stream turned a
  display-quality copy into something that could pick a conversation. Removing the
  readers first, in PR 2, makes PR 3 a pure deletion.

## Old runner history: option C, drop

**User decision, 2026-09-25: C.** Design options were:
- **A:** explicit `--file` read plus a hint;
- **B:** a labeled fallback and an `--include-legacy` search flag;
- **C:** drop.

The design recommended A. The user chose C. This supersedes the earlier
[native-session-identity](native-session-identity.md) wording that existing files
"stay as labeled legacy evidence".

- There is no runner-history decoder or legacy reader module.
- An unbound chat reports `unbound`, with no legacy hint.
- `session log --file <history.jsonl>` errors "not a native transcript".
- Search is native only.
- An archive ZIP with a legacy `history.jsonl` member restores as bytes only;
  `session log` does not read it.

**What C costs.** On this machine, about 155 chats (~850 MB) exist only as runner
history. They are unbound chats, mostly Pi runs from before Pi identity was tracked:
- 55 chats (62 MB) in meridian-cli;
- 100 chats (789 MB) in meridian-flow.

Those files stay on disk as inert JSONL. The archive ZIPs that might have held other
copies had already been reclaimed. Chats bound by the
[one-time import](legacy-native-import.md)
read their native files, so C loses nothing for them.

**Why not A or B.** The unique data is small and old. A keeps a decoder and a hint
path alive for it. B also adds a flag and an unverifiable second read path. Under C
the reader side of `history_codec` is deleted outright.

### Related retention decisions (same day)

- **Redundant runner history for bound chats** (~20 GB): the user said yes to a
  later `session archive` rule: when a chat is bound, its native file resolves and
  the run is older than N days, drop the run's `history.jsonl`. This is PR 3 scope.
- **Claude's 30-day transcript cleanup.** Claude's default `cleanupPeriodDays`
  deletes native files; the oldest remaining Claude file was 2026-08-26.
  - **User direction:** a regular archive of all conversations, run from cron,
    using the user's existing archive script outside Meridian.
  - **Tech lead's choices, not confirmed by the user:** the cadence (an hourly
    check that snapshots when the newest snapshot is ≥ 7 days old) and leaving
    `cleanupPeriodDays` unchanged.
  - **Consequence:** once Claude deletes a file and no archive restores it, that
    chat reports `missing`. A manual reclaim of archived originals has the same
    effect on reads.

## Search: a native-keyed trigram projection, verified exactly

Search keeps its semantics: a case-insensitive substring match over each entry's
whitespace-normalized text, with setup placeholders excluded. It stops parsing
transcripts at query time.

A disposable SQLite FTS5 projection, `native-search-v<N>.sqlite3`, indexes the
output of the same normalizer `session log` uses, keyed by native key rather than by
chat; chats are joined at query time from the authoritative bindings. The schema,
nomination, exact-verification and freshness-witness mechanics are in [native
transcript reads](../architecture/native-transcript-reads.md#search-projection).

So the projection cannot bind a chat, keep a chat that was unbound, or decide which
source a chat names.

**Why this shape.** Measured on the meridian-cli corpus (2,149 bound chats; design
`pr2-native-reads.md` §2.1):

| Option | Why rejected or chosen |
|---|---|
| Parse at query time, as 0.6.7 did | The corpus takes 12.7 s to parse; a 2 s budget always truncates. 0.6.7 was also slow because it resolved every candidate (~300 ms per OpenCode session). |
| Raw-byte token prefilter | Weak nomination (137 files for 3 true hits); every file for common or non-ASCII words |
| Lean extractor as the index builder | Produces different text from `session log`, so open commands would point at the wrong ordinals |
| FTS5 `unicode61` | Token match, not substring: wrong semantics |
| FTS5 trigram with content stored in FTS | 756 MiB |
| FTS5 trigram over raw display text | Misses matches: SQLite's folding leaves 391 codepoints unmapped, the tokenizer stops at NUL, and phrases that span a newline miss |
| **Chosen: contentless trigram, `detail=none`, over Python-folded, whitespace-collapsed, NUL-free text; exact re-check** | 0 misses across 419,253 true matches in the probe; ~200 MiB; one changed source costs one reparse |

This reverses the earlier "SQLite FTS rejected" line in the
[history-storage decisions](history-storage.md#d-history-file-authority). That
rejection was aimed at FTS as transcript *storage* next to runner files. Here the
projection stores nothing unique: its rows are a pure function of the bound keys,
the native bytes and the parser version.

**Accepted trade-offs** (the tech lead's calls on R2b, R3 and the reviews, 2026-09-25):
- **Bindings come from the authority, not the metadata index.** `native_bindings()`
  inverts the authoritative session fold with `session_fold.by_native_key` (about
  0.32 s warm). Reading the metadata index's `sessions` table instead measured about
  0.25 s faster. It was rejected because it adds a cold metadata dependency and alias
  tie-break rules, and it still would not reach the < 1 s target.
- **What `complete` means.** `complete` is true when every bound in-scope source was
  searched and the 100-match cap did not truncate. Sources that were searched but carry
  renderer warnings are a count line, with detail in `--json`; they never make a result
  incomplete. Unsearched sources do, and text output collapses them per reason class.
  This replaced R2b's output, where `complete` could never be true on real data and
  each search printed ~91 reason lines (alignment review S1).
- **Cold build order.** Unindexed keys are ordered by chat start time until a
  native locator is known. Strict native-mtime order would need a harness-owned
  bulk exact resolver; for a one-time cold build that is not worth it. An
  ops-side filename scan would break the exact-source boundary.
- **`session index rebuild` builds search and leaves previews lazy.** The old eager
  preview-warming loop re-folded authority and re-resolved every chat; keeping it
  would multiply rebuild cost. Browse refreshes previews on demand.
- **WAL with `synchronous=NORMAL`.** Each source's replace is one atomic
  transaction, so a process crash loses at most the source being written. Only
  power-loss durability of a rebuildable cache is weaker. Full sync made the
  build ~183 s.
- **Sources with warnings stay searchable.** OpenCode sources with read warnings
  but `search_ready=True` (87 here, thousands of hits) remain searchable.
- **Remaining questions answered by the tech lead:**
  - setup entries stay in search for PR 2;
  - untracked raw-ID lookup stays;
  - the first cold search may take up to 15 s and then prints its coverage;
  - loose spawns with no native source are accepted;
  - ~200 MiB of projection is fine.

**Measured on copies of this project's runtime** (V2, `evidence/pr2-v2-measure.md`;
2,037 keys):

| Case | Installed (PR 1 code) | PR 2 |
|---|---|---|
| Warm search, 7 queries | 2.8–3.1 s, truncated by its scan budget | 1.23–1.77 s |
| Hit sets vs a brute-force full native parse | — | equal on 7 of 7 queries (0 missing, 0 extra) |
| Cold first query | — | 15.9 s, with a coverage line |
| Full `session index rebuild` | 985 s, prewarming 1,356 previews | 55 s |

**The < 1 s warm target is missed, and the tech lead accepted the miss for PR 2.** The
query path is about 0.44 s:
- bindings 0.32 s;
- witnesses 0.05 s;
- FTS plus exact verification 0.05 s.

The rest is the import and startup cost of `meridian session …`: 0.75 s on the installed
PR 1 build and 0.85 s on PR 2. It predates PR 2, and PR 2 adds about 0.01 s. That cost
is CLI-wide and is tracked as #527. The target came from the design, not the user; no
user answer has yet accepted or waived it.

Also rejected during R2b:
- **One SQL `OR` term per bound key for scoped search.** SQLite raised "Expression
  tree is too large" at ≥ 1,000 keys. Scopes are set-based.

## The metadata index calls the shared fold; it never folds keys itself

The metadata index (`history.sqlite3`) keeps its discovery duties, but no longer
reads runner-history contents:
- **Activity** comes from lifecycle timestamps.
- **`read_targets`** is deleted: under C nothing asks the index which transcript to
  read.

If the index projects sessions, it must apply the authority's key rules. R3's
reproduction on synthetic roots showed what goes wrong otherwise. The index started
each generation from an empty record, so:
- a key-less resume generation lost its key;
- a conflicting start created a generation.

The authoritative fold does neither.

**Decision (authorized 2026-09-25).** Extract the per-event generation step into
`state/session_fold.py` as a pure `project_session_generation(...)`, and make
`fold_session_generations` loop over it. The index persists the fold's working state
and feeds only appended events through the same step. Gates:
- the 1,000-seed generated equivalence test is unchanged;
- the copied-journal output is byte-equal;
- `session_fold.py` stays under 500 lines;
- `history_index.py` shrinks.

**Landed (R3).** Metadata index `SCHEMA_VERSION` 6 persists two working sets: the
generation rows and `session_chats`, the accepted records, with the blank-generation
pointer as a column. The index feeds appended events through
`project_session_generation`. Results:
- `session_fold.py` is 487 lines, and `history_index.py` went from 1,700 to 1,473.
- On a copy, index keys equal the authority's: 7,113 of 7,113 generations, and all
  7,041 chat records.
- Rebuilding with every `history.jsonl` deleted gives the same `records`, `sessions`,
  `aliases` and `session_chats`.
- One chat, `c248`, leaves the primary list. The old index had indexed a conflicting
  primary start that the authority rejects, so this is a correction.

Rejected:
- **Letting the index fold keys its own way.** It drifts from the authority in
  exactly the cases above.
- **Copying the fold loop into the index.** That duplicates the binding policy the
  restructure had just reduced to one place.
- **Re-folding the whole journal on every refresh.** About 0.4 s per command.

## Run facts come from what the attempt saw

Usage, failure, "produced output" and the first session ID come from an in-memory
fold over the events the runner already received live: one per-harness `AttemptFold`,
holding one `AttemptFacts`, per attempt. The report comes from the first source that
has one:
1. an explicit `report.md`;
2. a Pi typed failure;
3. the exact native reply named by this attempt's events (OpenCode V2 only);
4. the fold's final text;
5. the failure reason.

Otherwise the value is unknown (`None`, never zero).

The runner no longer re-reads its stream after exit. "The last assistant message
after the attempt started" is never used: a time window is not attribution. On a copy,
200 real finished runs replayed through the fold matched the old extraction with 0
unexplained differences. The sample held no Pi runs, so Pi rests on a fake-harness
differential and real shell-shim tests.

**Incomplete facts never erase a harness total** (recheck NF1, fixed in lane D). A fold
exception or a malformed captured line marks the facts `incomplete`, which is logged.
- The first fix nulled all usage on `incomplete`. One `Warning:` line on a Claude
  `--print` stdout then dropped the harness-reported `total_cost_usd` and blinded budget
  enforcement. That is a cost regression.
- **Now:** only generic fallback usage is dropped, and only after a fold step fails.
  Harness-specific totals survive, and a malformed line alone drops nothing.
- **Why:** a partial generic sum must not pass as a total, but a harness's own final
  total is not partial.

To make this possible, event delivery stops depending on the history writer.
Before, observer dispatch ran only after a successful history write, and managed
primary attach raised when it had no writer. One emit path now runs three steps in
order: inline hooks, then the optional write, then subscriber fan-out. The unused,
lossy observer registry was deleted.

**Bounded replacements for stream readers:**
- **Pi lifecycle phases.** `spawn show` now reads a per-spawn
  `pi-lifecycle.json` sidecar: last phase and cleanup status per attempt. Before, it
  scanned `history.jsonl`.
  - Pi's harness bundle declares it as an event sink, written only when a phase event
    arrives. The manager and attach carry no Pi branch.
  - The read-modify-write is lifetime-locked with the spawn row, so a late phase
    cannot recreate a deleted spawn.
  - Sinks outlive teardown, so cleanup phases survive shutdown.
- **Staleness.** The reaper and the read-only stale check drop the "fresh history
  mtime means alive" leg. Heartbeat, now also touched by managed primary attach,
  plus process and report evidence remain.
- **Transcript hint.** `spawn show`'s session-log hint appears only when the
  continue chat has a native key.
- **Guardrail env.** `_MERIDIAN_GUARDRAIL_OUTPUT_LOG` is removed. It was documented
  as `output.jsonl` but actually pointed at `history.jsonl`. Guardrail scripts get
  `_MERIDIAN_GUARDRAIL_REPORT` (the `report.md` path) and `_MERIDIAN_GUARDRAIL_CHAT_ID`
  (for `meridian session log`) instead.
- **Empty-output failures** no longer write an artifact into the history path.

## PR 3 is pure deletion, and a test mode proves it

`pytest --runner-history=off` makes writers absent rather than silent: no
`HarnessHistoryWriter` is constructed. Any read of a spawn, attempt or artifact
`history.jsonl` raises. Every failure in that mode is classified:
- a test of the runner-history format goes on PR 3's deletion list;
- a user-visible fact that still depends on the stream blocks PR 2.

**Subprocesses are covered by a lazy import hook.** A test-only `sitecustomize` installs
the read trap and a meta-path hook that patches the writer modules when they are
imported.
- **Rejected:** gating the patch on `sys.argv[0]`. Under `python -m meridian`,
  `argv[0]` is `'-m'` at site time, so CLI subprocess tests silently ran with real
  writers ([lesson](../lessons/native-session-identity.md#an-argv-gate-in-sitecustomize-misses-python--m-children)).
- **Rejected:** importing Meridian eagerly in every child. It slows unrelated
  children.

A test asserts that `python -m meridian` children see absent writers.

**Result at `3ae3fce8`:** 2,166 passed and 16 failed. Each failure was checked by name
against PR 3's list, and all 16 are writer tests. No trap fires in a production frame.

PR 3 then deletes (`evidence/pr3-deletion-list.md`):
- writer construction and every write, including `_emit`'s `Written` and
  `WriteFailed` arms;
- the retry `meridian.attempt.completed` marker and the direct header write;
- `write_retained_child_stream` and `ingest_portable_history`;
- `last-observed-event.json` and the reaper's diagnostic read of it;
- the dogfood-row translator;
- the 16 writer tests and the F1b differential oracle (`tests/support/legacy_f1b/`).

It also adds the approved archive rule. When a chat is bound, its exact native source
resolves and the run is older than N days, the run's `history.jsonl` is dropped. A
search-index row is not evidence that the native source exists.

`history_codec` is not deleted wholesale: its portable header types also serve sealed
native snapshots.

## Consequences

- **Old `pN` refs.** An old spawn ref with a bound chat now shows the chat's whole
  native conversation, labeled, not that run's events alone. For a primary resumed
  across many runs, that includes the other runs. When the chat is unbound, the error
  names the chat, not the spawn.
- **Archives.** Archived chats keep their binding and native file, so corpus search
  includes them, and search's `--include-archives` is removed.
  `session browse --include-archives` stays, as a filter that adds archived rows to
  the picker (V1's call; it filters rows and no longer selects a read source).
- **Result caps.** Search no longer truncates because a time budget ran out, and
  every hit is verified exactly. Common words still fill the 100-match cap with the
  newest sessions' setup text, because Codex embeds AGENTS.md in every session. A
  capped result is reported as truncated, so it is not `complete`.
- **Disk.** A project keeps two index files, both disposable: the metadata index
  (174 MB here) and the search projection (218 MB).
- **Unarchivable spawns.** Spawns with no native source stay loose indefinitely:
  unbound spawns, launches that failed before a session existed, and bound chats
  whose native file is gone. Archive capture needs a transcript member.
- **Crashed runners.** `AttemptFacts` lives in memory. A crashed runner is
  finalized from `report.md` and lifecycle facts only.

**Provenance:** `work:native-harness-session-identity`:
- design: `design/pr2-native-reads.md`, revision 3 (design `spawn:p7105`; reviews
  `spawn:p7112`, `spawn:p7113`, `spawn:p7115`);
- `decision.md` entries of 2026-09-25: "User decisions on old data + archive
  cron", "R2b done; its decisions accepted", "R3 escalation → fold extraction
  authorized", and "USER DECISION … three stacked PRs";
- `decision.md` entries of 2026-09-25 from "R3 + F1b done" through "Codex harness auth
  failure → Claude backups" (S1, the < 1 s acceptance, NF1, #527);
- `DIVERGENCE/pr2-r2b-cost-and-rebuild.md`;
- lanes R1 `spawn:p7117`, R2a `spawn:p7118`, F1a `spawn:p7119`, F2 `spawn:p7123`,
  A1 `spawn:p7124`, R2b `spawn:p7125`/`spawn:p7127`, R3 `spawn:p7128`/`spawn:p7130`,
  F1b `spawn:p7131`, V1 `spawn:p7139`;
- reviews: thermo `spawn:p7140`, alignment `spawn:p7141`, recheck `spawn:p7147`;
  fix lanes A `spawn:p7142`, B `spawn:p7146`, C `spawn:p7145`, D `spawn:p7151`;
- V2 measurement `spawn:p7148`/`spawn:p7152` (`evidence/pr2-v2-measure.md`);
- search-cost probe `spawn:p7101`;
- conversation `chat:c6945`.
