# Decision: Conversation history is native-only

**Status: settled 2026-09-25.** User decisions: three stacked PRs; old runner history
option C (drop). Reads, search and run facts move off runner `history.jsonl` in
**PR 2** (branch `feat/native-reads`, stacked on PR #520). That PR is still being
built, and each section below says what is merged on the branch and what is only
designed. **PR 3** stops writing the stream. How the reads work:
[native transcript reads](../architecture/native-transcript-reads.md). The identity
rule this builds on: [native session identity](native-session-identity.md).

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

The user's framing (2026-09-25): "stop relying on history.jsonl for anything … i
don't think claude even relies on history.jsonl", and switching mid-session "looks
like it will break history.jsonl and is just extra processing that we are basically
wasting".

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
[one-time import](native-session-identity.md#chats-from-before-the-key-existed-import-once)
read their native files, so C loses nothing for them.

**Why not A or B.** The unique data is small and old. A keeps a decoder and a hint
path alive for it. B also adds a flag and an unverifiable second read path. Under C
the reader side of `history_codec` is deleted outright.

### Related retention decisions (same day)

- **Redundant runner history for bound chats** (~20 GB): the user said yes to a
  later `session archive` rule: when a chat is bound, its native file resolves and
  the run is older than N days, drop the run's `history.jsonl`. This is PR 3 scope.
- **Claude's 30-day transcript cleanup** (Claude's default `cleanupPeriodDays`
  deletes native files; the oldest remaining Claude file was 2026-08-26). The user
  did not change `cleanupPeriodDays`. Native stores are backed up by the user's own
  scheduled archive script, run by cron outside Meridian. Consequence: once Claude
  deletes a file and no archive restores it, that chat reports `missing`.
  Meridian's reclaim of archived originals has the same effect on reads.

## Search: a native-keyed trigram projection, verified exactly

Search keeps its semantics: a case-insensitive substring match over each entry's
whitespace-normalized text, with setup placeholders excluded. It stops parsing
transcripts at query time.

A disposable SQLite FTS5 projection, `native-search-v<N>.sqlite3`, indexes the
output of the same normalizer `session log` uses. Rows are keyed by native key, not
by chat. Nomination and verification:
- The index text is case-folded by Python.
- The trigram `MATCH` nominates candidate entries.
- Every candidate is re-checked with today's Python predicate.
- Chats are joined at query time from the authoritative bindings.

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

**Accepted trade-offs** (the tech lead's calls on R2b, 2026-09-25):
- **Correctness does not depend on the metadata index.** R2b reads bindings through
  one `native_bindings()` seam over the authoritative session fold.
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

**Measured on a copy** (2,016 keys):

| Case | Time |
|---|---|
| Warm p50 | 1.20–1.73 s, complete |
| Installed 0.6.7 | 2.83–3.06 s, truncated |
| Cold first query | ~15.8 s, with a coverage line |
| Full rebuild | 65 s |

The warm time still misses the design's < 1 s gate. Folding bindings costs 0.39 s
of it, and a time breakdown after R3 decides what to optimize.

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

Rejected:
- **Letting the index fold keys its own way.** It drifts from the authority in
  exactly the cases above.
- **Copying the fold loop into the index.** That duplicates the binding policy the
  restructure had just reduced to one place.
- **Re-folding the whole journal on every refresh.** About 0.4 s per command.

## Run facts come from what the attempt saw

Report, usage, failure class, "produced output" and the first session ID come from
the first source that has them:
1. an explicit `report.md`;
2. an in-memory fold over the events the runner already received live, one
   `AttemptFacts` per attempt;
3. a native turn that this attempt's own events named;
4. otherwise unknown (`None`, never zero).

The runner no longer re-reads its stream after exit. "The last assistant message
after the attempt started" is never used: a time window is not attribution.

To make this possible, event delivery stops depending on the history writer.
Before, observer dispatch ran only after a successful history write, and managed
primary attach raised when it had no writer. One emit path now runs three steps in
order: inline hooks, then the optional write, then subscriber fan-out. The unused,
lossy observer registry was deleted.

**Bounded replacements for stream readers:**
- **Pi lifecycle phases.** `spawn show` now reads a per-spawn
  `pi-lifecycle.json` sidecar: last phase and cleanup status per attempt. An inline
  hook writes it atomically, and only when a phase event arrives. Before, it
  scanned `history.jsonl`.
- **Staleness.** The reaper and the read-only stale check drop the "fresh history
  mtime means alive" leg. Heartbeat, now also touched by managed primary attach,
  plus process and report evidence remain.
- **Transcript hint.** `spawn show`'s session-log hint appears only when the
  continue chat has a native key.
- **Guardrail env.** `_MERIDIAN_GUARDRAIL_OUTPUT_LOG` is removed. It was documented
  as `output.jsonl` but actually pointed at `history.jsonl`. Guardrail scripts get
  the `report.md` path and the chat ID instead.
- **Empty-output failures** no longer write an artifact into the history path.

## PR 3 is pure deletion, and a test mode proves it

`pytest --runner-history=off` makes writers absent rather than silent: no
`HarnessHistoryWriter` is constructed. Any read of a spawn or artifact
`history.jsonl` raises, and subprocess tests inherit the trap through a test-only
`sitecustomize`. Every failure in that mode is classified:
- a test of the runner-history format goes on PR 3's deletion list;
- a user-visible fact that still depends on the stream blocks PR 2.

PR 3 then deletes:
- writer construction and every write;
- the retry `meridian.attempt.completed` marker;
- the direct header write;
- `write_retained_child_stream`;
- `last-observed-event.json`;
- the dogfood-row translator.

## Consequences

- **Old `pN` refs.** An old spawn ref with a bound chat now shows the chat's whole
  native conversation, labeled, not that run's events alone. For a primary resumed
  across many runs, that includes the other runs.
- **Archives.** `--include-archives` goes away: archived chats keep their binding
  and native file, so they are searched by default.
- **Result caps.** Search results are complete, not truncated. Common words fill
  the 100-match cap with the newest sessions' setup text: Codex embeds AGENTS.md in
  every session.
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
- `DIVERGENCE/pr2-r2b-cost-and-rebuild.md`;
- lanes R1 `spawn:p7117`, R2a `spawn:p7118`, F1a `spawn:p7119`, F2 `spawn:p7123`,
  A1 `spawn:p7124`, R2b `spawn:p7125`/`spawn:p7127`, R3 `spawn:p7128`;
- search-cost probe `spawn:p7101`;
- conversation `chat:c6945`.
