# Decisions: Session Browse Picker

Each row of `meridian session browse` shows what a person recognizes a session by
(tmux target, agent and model, time since last turn, work item), and a transcript
appears only when asked for. The owner set the columns and the interaction; the
data behind them gives each fact one source.

**Status:** settled 2026-10-21 (`work:session-browse-redesign`); shipped in
meridian-cli PR #714 (merge `471dfbc59`, 2026-10-21).

Related records: [prompt_toolkit as the framework](tui-framework.md),
[bare `--continue` opens the picker and the Resume/Fork/Blocked decision](launch-session-initiation.md#session-browse-2026-08-session-browse-design),
[lineage as conversation identity](state.md#d-lineage-conversation-identity),
[`--from` resolved at the request boundary](launch.md#d-from-request-boundary).
Current mechanism: [session operations](../codebase/session-operations.md#session-browse).

## D-browse-recognition-row: rows show what you recognize a session by {#d-browse-recognition-row}

**Decision (owner).** A row reads, left to right:

```
 Status: [Active] Archived  Sort: [Updated] Created                50 of 309
> type to search
    TMUX            AGENT                        IDLE  WORK
▸   meridian:main   tech-lead (claude-opus-5-5)    3m  session-browse-redesign
  ⚠ research:main   solo-dev (gpt-6-astra)         2h  pr-conflict-refresh
enter resume  space view  l lineage  tab filters  / transcripts  esc quit
```

- **`⚠`** when the session's prompt cache has probably expired. The provider
  caches the conversation prefix for a few minutes to an hour; after that, the next
  turn pays to process the whole conversation again, so the icon tells you whether
  re-entering is cheap. No explanatory text: idle time and icon together are enough
  ("I would just know it").
- **TMUX** is `session:window`, captured at launch. It is blank for a session
  started outside tmux, or before the capture existed.
- **AGENT (model)** is one column. The model is the last one the session ran, not
  the launch model.
- **IDLE** is time since the last turn. A session running for hours no longer looks
  as old as its launch.
- **WORK** comes last because it is the column that may be cut off. On narrow
  terminals WORK shrinks first, then `(model)` drops, then AGENT.

**No chat id in rows**, including `--plain`. The chat id appears in the detail
header and still matches type-to-search. The owner: "I don't think c-id is
important upfront", and later "I don't think we need the c-id in the initial table
either".

**No live marker, no live-first sort.** Running and stopped sessions sort together
by last turn, and `⚠` applies to running rows too. Whether a session is running
matters only to what Enter does: a running row forks (it is never attached twice),
a stopped row resumes. The footer names that action (`enter fork`, `enter resume`).
The owner: "I don't think we should care about "live" or not".

**Rejected:** a split preview pane under the list. At ordinary terminal heights it
can show only the tail of a few messages, which is why the old pane almost always
read "clipped".

**Prior art:** `codex resume` (title and relative age, id hidden, filter and sort on
the search line, always-on search) is the picker the owner asked this one to
resemble. Claude Code's Space-to-preview supplied the detail key. None of the
surveyed pickers show agent, model, work item or cache state; those columns are
Meridian's own.

## D-browse-transcript-on-demand: the transcript opens on request; the preview subsystem is gone {#d-browse-transcript-on-demand}

**Decision (owner).** Moving through the list reads no transcript. Space opens a
full-screen detail view: a header with chat id, harness, tmux, work, agent, model,
created time and last turn, over the whole transcript. The view parses the
transcript once, off the UI thread, scrolls within that snapshot, and reloads only
when the session's generation changes. The owner: "the "preview" should be like a
command to enter instead of loading automatically and cutting it off".

The bounded preview projection is deleted: its accumulator, its SQLite cache table,
its rebuild reporting and its offline-preview labels. The detail view and
`session log` share one reader that windows an already-parsed transcript, so there
is still exactly one transcript interpretation path. This supersedes
[D-history-bounded-preview](history-storage.md#d-history-bounded-preview-previews-are-disposable-projections-of-canonical-history-2026-09-14).

**Why delete it rather than enlarge it.** A preview that loads on every highlight
must be bounded to stay fast, and a bounded preview is clipped by construction.
Once the transcript opens on request there is no hot path left for the cache to
accelerate, and the cache carried its own bugs: on chat rows it labelled the
transcript with another run's identity, and it kept a failed lookup's stale status.

## D-browse-staleness-from-idle-state: last turn and cache staleness need no new durable data {#d-browse-staleness-from-idle-state}

**Decision.** "Updated" order and IDLE use facts that already exist:

- A **stopped** session's last turn is its `stopped_at`.
- A **running** session's last turn comes from its idle file, which the idle
  notifier already keeps ([idle adapters](../architecture/idle-adapters.md)).
  Running sessions are placed in the order by that time.
- **`⚠`** is computed at render time from a cache deadline: last turn plus the TTL
  the idle file recorded, or a per-harness default (Claude, OpenCode and Pi 300 s;
  Codex 1800 s). The row stores the deadline, never a frozen "stale" flag, so a
  picker left open turns rows stale as their deadlines pass.
- A running session with **no idle file** has an unknown deadline and shows no `⚠`.
  Unknown is not the same as fresh, and guessing would put a false warning on
  harnesses without an idle sensor.

Idle files are notification state with a 7-day garbage collection, so a missing
file is expected and falls back to stop or start time.

**Rejected:** appending a turn event to the session journal on every turn. It
would add a write per turn to every session, for a fact that the stop record and
the idle file already cover.

## D-browse-query-before-limit: one matcher, applied before the row limit {#d-browse-query-before-limit}

**Decision.** The Status (Active/Archived) and Sort (Updated/Created) controls and
the type-to-search text are inputs to one `session_list` query, which filters and
orders in the [history index](../architecture/state-system/session-state.md)
(a disposable SQLite projection of the session journals) **before** `LIMIT` and returns the count for that scope.
That query is the only definition of a match. The TUI never matches filter text
against rows itself; typing re-runs the query after a short debounce and shows
`loading…` until the answer for the current scope lands. Matching is by substring
over tmux target, work, agent, model, chat id and cwd.

**Why.** The old picker cut to 50 rows first and filtered in the TUI, so a search
for a chat outside the first page reported `0 of 309`. Moving the match into the
query left the TUI filter in place for a while, and two matchers drift; the one that
sees only the loaded page is wrong by construction, so it was deleted.

`/` remains transcript search over the rows the Status view shows. The old
`--include-archives` flag is accepted with a warning and has no effect, for one
minor release ([compatibility policy](compatibility-versioning.md)), because Status:
Archived replaced it.

## D-browse-lineage-view: lineage is top-level `--fork` and `--from` ancestry of primary sessions {#d-browse-lineage-view}

**Decision (owner).** `l` (Ctrl-L once filter text exists) opens a tree of the
highlighted session's ancestors and descendants through `--fork` and `--from` edges,
each node showing chat id, edge kind, tmux, agent, idle time and work. Subagent spawns are
not part of it. The owner: "not hte subagent lineage but just the --from top-level
lineage, and then be able to just see the ids".

So the tree covers primary sessions only. The history index writes lineage edges
only for primary chats, and a spawn's `--from` is not recorded as a session parent:
nothing could read it there. A spawn's `--from` remains visible in its create output
and its `starting-prompt.md`. Recording it would pay off only if spawns became browse
rows, which is a product change.

The edges themselves are conversation identity, carried across continuation: see
[D-lineage-conversation-identity](state.md#d-lineage-conversation-identity).

## D-browse-python-first: rewrite in Python; Rust waits for the data layer {#d-browse-python-first}

**Decision (owner).** The picker was rewritten in Python on prompt_toolkit. The
owner had asked whether a Rust TUI would be faster; after the profile below:
"ok, python first still".

**Evidence.** Before the rewrite, `session browse --plain --limit 50` took 1.0 s and
110 MB. About 0.65 s was imports of unrelated session operations pulled in at CLI
load, and about 0.39 s was the managed-session work lookup folding the journal.
Drawing was negligible. A Rust frontend would need its own copy of the state model
or would shell out to Python for the same reads.

**Result.** Handler-local imports, one lazy command-registration path, and an
indexed work lookup brought `--plain` to about 0.5 s and 81 MB, with first paint at
0.48–0.60 s. The original 200 ms target is **not met**. What remains is shared by
every command: the CLI's base imports (about 175 ms) and SQLAlchemy plus Pydantic
setup on the state read path (about 160 ms). That gap is tracked in meridian-cli
`src/meridian/cli/.context/TODO`, which also forbids a parallel fast CLI loader.

**Revisit when:** the state read path no longer needs SQLAlchemy, or the
[Rust rewrite](rust-rewrite.md) moves the data layer, at which point a native
picker stops duplicating Python state.

## Follow-ups

- **Start a new session from a row with `--from`** (owner idea, settled as a stacked
  follow-up PR): Enter-alternative that launches `meridian --from <id>` with an
  optional extra prompt, instead of continuing.
- **Cross-project "All" scope.** The listing reads one project's runtime root;
  tracked in meridian-cli `src/meridian/cli/.context/FUTURE`.
- **#558** asked for tmux labels (delivered here) and a recently-crashed promotion
  group (still open; see
  [idle notifications](idle-notifications.md)).

## Sources

`work:session-browse-redesign` (`decisions.md`, `plan.md` "Revision",
`findings.md`, `pr-body-session-browse.md`); owner rulings in `chat:c13711`;
picker research `spawn:p14094`, `spawn:p14095`; plan review `spawn:p14100`;
maintainability reviews `spawn:p14130`, `spawn:p14149`; live TUI probes
`spawn:p14123`, `spawn:p14131`, `spawn:p14150`.
