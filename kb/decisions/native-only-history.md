# Decision: Conversation history is native-only

**Status: settled 2026-09-25; PR 2 and PR 3 landed on their branches.** User
decisions: three stacked PRs; old runner history option C (drop).
- **PR 2** moves reads, search and run facts off runner `history.jsonl`. It is draft
  PR #526 (`feat/native-reads` @ `3ae3fce8`, stacked on PR #520), not on `main`. Every
  slice is merged: R1, R2a, R2b, R3, A1, F1a, F1b, F2 and V1. It went through a
  thermo-nuclear review, a design-alignment review, fix lanes A–D, and a recheck that
  passed. Before/after measurement on runtime copies is done.
- **PR 3** stops writing the stream and adds the prune rule. It is branch
  `feat/stop-runner-history` @ `c1fa08e4`, stacked on #526, not on `main`. Its review
  passed with fixes, and fix lanes E, F and G are merged. See [PR 3: the stream is
  deleted](#pr-3-the-stream-is-deleted).

How the reads work: [native transcript reads](../architecture/native-transcript-reads.md).
How run facts are computed and delivered: [attempt facts and
delivery](../architecture/attempt-facts-and-delivery.md).
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
- **The run stream is gone.** Since PR 3, Meridian does not write
  `spawns/<id>/history.jsonl` or its `last-observed-event.json` checkpoint. Live event
  delivery to subscribers stays in memory. The few facts that outlive the run are
  written as bounded files (`report.md`, `pi-lifecycle.json`, the spawn row); they are
  not a stream.

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
  `session archive` rule: when a chat is bound, its native file resolves and the run
  is older than N days, drop the run's `history.jsonl`. PR 3 implements it as an
  explicit, dry-run-first action; see [the prune rule](#the-prune-rule).
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
2,037 keys): the full before/after table is in [native transcript
reads](../architecture/native-transcript-reads.md#search-projection).

**The < 1 s warm target is missed, and the tech lead accepted the miss for PR 2.** The
query path itself is about 0.44 s (bindings, witnesses, FTS plus exact verification —
breakdown in the architecture page above); the rest is the import and startup cost of
`meridian session …`, which predates PR 2 and is tracked as #527. The target came from
the design, not the user; no user answer has yet accepted or waived it.

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
fold over the events the runner already received live, one per-harness `AttemptFold`
per attempt, with a fixed report-source precedence. The fold, `AttemptFacts` and the
precedence order are in [attempt facts and
delivery](../architecture/attempt-facts-and-delivery.md#attempt-folds). Otherwise the
value is unknown (`None`, never zero).

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

To make this possible, event delivery stopped depending on the history writer. Before,
observer dispatch ran only after a successful history write, and managed primary
attach raised when it had no writer. PR 2 made the write optional and deleted the
unused, lossy observer registry; PR 3 deleted the write. The emit path is now inline
hooks, then subscriber fan-out, then the coordinator's `note_event_delivered`
([detail](../architecture/attempt-facts-and-delivery.md#the-emit-path)).

**Bounded replacements for stream readers.** Pi lifecycle phases move to a per-spawn
sidecar instead of a `history.jsonl` scan; the reaper drops its "fresh history mtime
means alive" leg; the transcript hint and the guardrail env vars stop naming
`history.jsonl`. (`_MERIDIAN_GUARDRAIL_OUTPUT_LOG` was documented as `output.jsonl` but
actually pointed at `history.jsonl`; guardrails now get the report path and chat ID.) empty-output failures no longer write into the history path. Each
replacement is detailed in [attempt facts and
delivery](../architecture/attempt-facts-and-delivery.md#other-stream-readers-removed).

## PR 3: the stream is deleted

**A test mode proved PR 3 could be pure deletion.** In PR 2, `pytest
--runner-history=off` made the writers absent and trapped every read of a spawn,
attempt or artifact `history.jsonl`. Every failure in that mode was classified: a test
of the runner-history format went on PR 3's deletion list; a user-visible fact that
still depended on the stream blocked PR 2. At `3ae3fce8` the suite failed only on the
16 writer tests, and no trap fired in a production frame.

Subprocesses were reached by a lazy import hook in a test-only `sitecustomize`.
- **Rejected:** gating the patch on `sys.argv[0]`. Under `python -m meridian`,
  `argv[0]` is `'-m'` at site time, so CLI subprocess tests silently ran with real
  writers ([lesson](../lessons/native-session-identity.md#an-argv-gate-in-sitecustomize-misses-python--m-children)).
- **Rejected:** importing Meridian eagerly in every child. It slows unrelated
  children.

**What PR 3 deleted** (`evidence/pr3-deletion-list.md`; every row is gone and the
writer-symbol grep over `src/` and `tests/` is empty):
- `state/history.py` as a whole: the writer, envelopes, tail repair, causal
  rehydration, the `last-observed-event.json` checkpoint, `write_retained_child_stream`
  and `ingest_portable_history`;
- the managed-primary causal tracker, which only the writer used;
- the writer registry, `EmitOutcome`, and the drain loop's write-failure abort;
- the retry header and the `meridian.attempt.completed` marker;
- the reaper's `last_observed_event` evidence;
- the dogfood-row translator (replaced; below);
- the 16 writer tests, the F1b differential oracle (`tests/support/legacy_f1b/`) and
  `tests/support/history.py`.

Kept: the option-C `--file` rejection, legacy ZIP inventory and restore as bytes,
Claude's own `~/.claude/history.jsonl` lookup, Claude `--print` `output.jsonl`
capture, `pi-lifecycle.json`, and `history_codec`'s portable header types, which
sealed native snapshots use. Blind mode now keeps only its read trap; with no
writers left, the writer patches were retired.

### Dogfood rows migrate once, not on read

The PR 1 dogfood build wrote the run boundary as flat `entry_chat_id`,
`exit_chat_id`, `exit_identity` and `trampoline_successor_id` fields. PR 1's
restructure read them through a `model_validator` translator. At PR 3 there were 68
such rows on the author's machine, and the installed PR 1 build kept writing more.

**Decision (P3a, kept by the tech lead):** a one-time migration,
`state/spawn/dogfood_migration.py`, replaces the translator.
- It prefilters by bytes, then takes the shared mutation lock and the spawn lock,
  re-reads the row, translates it, validates it as `StoredSpawnState`, and rewrites
  it atomically. It is idempotent.
- Each row is isolated (review finding 1). A malformed row is reported with the
  failing field, for example `ValidationError: run_boundary.status: …`, and the pass
  continues.
- `meridian doctor` and the background repairs of every primary launch run it.
- Until it runs, those rows quarantine. `spawn show pN` ends the quarantine message
  with "run `meridian doctor` to migrate it", and prune and archive list the IDs
  with the same pointer.
- When it migrates at least one row, it re-arms a history-index `authority`
  initialization failure (lane F); see [history-index
  initialization](history-storage.md#d-history-index-initialization).
- Delete the module, `repository.DOGFOOD_BOUNDARY_FIELDS` and the re-arm call once no
  dogfood rows remain.

**Rejected:** an atomic rewrite at state load. `read_state` runs inside
`write_state_locked`, which holds the non-reentrant spawn lock, and inside many
read-only paths. An unlocked rewrite there could clobber a concurrent locked write.
**Rejected:** running the migration on every command's startup. It scans every
`state.json` per command unless a done-marker gates it, and a still-running PR 1
runner can write a new dogfood row after the marker is set.

### The prune rule

`meridian session archive --prune-runner-history [--apply] [--after-days N]`
(`ops/runner_history_prune.py`) drops runner-stream files that an exact native source
makes redundant. Mechanism: [session operations](../codebase/session-operations.md#runner-history-prune).
The decisions behind its shape:
- **Explicit only, dry-run first.** Automatic maintenance
  (`session_stop_maintenance`, `history.archive.automatic`) never calls it.
- **The default is 14 days**, measured from `terminal.finished_at`. The general
  archive default of 30 days does not apply.
- **Qualification mirrors `session log pN`.** Every native source `session log pN`
  would read must resolve exactly now, through `session_target.resolve_run_sources`.
  The search index is never evidence.
- **The entry chat must resolve too** (P3b, stricter than the user's wording). When a
  verified exit moved the run to another chat, the runner stream still covers the
  entry chat's part of the run.
- **A run boundary that is `unresolved` or `mismatch` is skipped** (`exit_unresolved`):
  the run may have ended in a native session nobody identified. A boundary of `None`
  (runs from before exit tracking) qualifies. Consequence: only Pi reports an exit
  identity, so every Claude, Codex and OpenCode row written by a PR 1-or-later build is
  skipped. On the copy that was 55 spawns (124 MB). New runs write no stream, so this
  set does not grow.
- **An unreleased live process scope is skipped.** Archive's active-chat and
  dependency protections are not imported: a terminal spawn's stream is final, and
  deleting it does not change ancestry.
- **Multi-attempt runs qualify.** Meridian reads no runner bytes for any attempt, and
  each attempt's turns stay in the harness's own files (review question 10, answered
  "no change").
- **Apply re-checks under the lock, per spawn.** It holds the archive lock for the
  pass. Each spawn is unlinked under the spawn aggregate lock only if its record is
  unchanged and its native sources still resolve. One spawn's failure is recorded and
  the pass continues.

**Measured on a copy of this project's runtime** (`evidence/pr3-archive-report.md`):

| `--after-days` | Prunable | Size | Main skips |
|---|---:|---:|---|
| 14 (default) | 123 spawns | 0.80 GB | 1,155 recent (4.1 GB), 22 unbound, 4 running |
| 0 | 1,190 spawns | 4.76 GB | 55 `exit_unresolved` (124 MB), 55 unbound (62 MB), 4 running |

After both applies, `session log`, `spawn show --json` and ref-scoped `session search`
were byte-equal for the sampled spawns, a rerun pruned nothing, and exactly the listed
files were removed. The dry run takes 3 s at 14 days and 13 s at 0 days, mostly Codex's
`rglob` over `~/.codex/sessions`. The user decides when to run it on a real runtime.

**Caveat: Claude deletes its own transcripts.** Claude's `cleanupPeriodDays` (default
30) removes old native files. Once pruned, a Claude chat has no runner copy, so after
Claude's cleanup it reads `missing`. Meridian never read the runner copy after PR 2, so
this loses only raw bytes for manual forensics. The user's weekly archive cron keeps
copies. On this project only 15 Claude spawns (13 MB) were affected.

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
- **Upgrading to PR 3.** Run `meridian doctor` once, or launch a primary, so dogfood
  rows migrate. Until then those rows quarantine, and an index reproject that hits
  them latches an `authority` failure that names the row and `meridian doctor`. The
  migration clears it, and no manual `session index rebuild` is needed.
- **Old runner files stay** until the user runs the prune rule. Files the rule skips
  (unbound, `exit_unresolved`, or with a missing native source) stay as inert bytes.

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
- PR 3: `evidence/pr3-deletion-list.md`; lanes P3a `spawn:p7155`
  (`evidence/pr3-p3a-report.md`), P3b `spawn:p7156` (`evidence/pr3-archive-report.md`),
  blind-test fix `spawn:p7160`; review `spawn:p7161` (`review/pr3-review.md`); fix
  lanes E `spawn:p7162`, F and G `spawn:p7163`/`spawn:p7164`
  (`evidence/pr3-fix-e-report.md`, `evidence/pr3-fix-f-report.md`); `decision.md`
  entries of 2026-09-26; code checked at `feat/stop-runner-history` @ `c1fa08e4`;
- conversation `chat:c6945`.
