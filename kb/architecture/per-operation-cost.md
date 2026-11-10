# Per-operation cost

Launch, close, wait, show, cancel and continue concern one spawn and one chat, so their work should not grow with project history. Global scans belong to explicit global commands or bounded maintenance. Measure physical reads and lock hold times across corpus sizes, not only wall time on a shared machine: a filtered pass still reads every byte ([contention diagnosis](../lessons/contention-diagnosis.md)).

A per-chat candidate index must be followed by authoritative reads: a stale or incomplete index can't authorize a lifecycle decision ([cost cuts and safety checks](../lessons/cost-cuts-and-safety-checks.md)).

## Where cost still grows with history

- **The session journal.** Every chat appends to one `sessions.jsonl` ([session state](state-system/session-state.md#sessionsjsonl-is-one-global-journal)). Per-operation reads go through one keyed seam with a byte prefilter, so they decode only the chats they need, but each pass still reads the whole file, and a primary close makes about ten such passes. A no-op foreground launch took 3.6, 6.0 and 9.1 s on 1, 19 and 47 MB journals. Per-chat journals are the chosen fix, not yet built ([#418](https://github.com/haowjy/meridian-cli/issues/418)). Bulk readers still outside the seam: listing all session records, listing generations, and the legacy native import.
- **Session teardown reads every spawn's state** ([#612](https://github.com/haowjy/meridian-cli/issues/612)).
- **History index.** Readers take the exclusive catch-up gate even when nothing changed ([#594](https://github.com/haowjy/meridian-cli/issues/594)); a cold build holds it for the whole build, 2.4–5.7 s ([#592](https://github.com/haowjy/meridian-cli/issues/592)); indexed spawn reads materialize every row, about 2 s at 12,000 spawns ([#600](https://github.com/haowjy/meridian-cli/issues/600)).
- **Doctor and lock GC scan everything** ([#601](https://github.com/haowjy/meridian-cli/issues/601)).
- **Memory.** Runner and primary processes hold hundreds of MB; one full materialization of spawns or sessions can set a process's lifetime RSS peak, because Python keeps the arenas ([#627](https://github.com/haowjy/meridian-cli/issues/627)).
- **A fixed floor.** A no-op launch costs about 3.6 s even with tiny history, from interpreter and subprocess startup ([#595](https://github.com/haowjy/meridian-cli/issues/595)).
