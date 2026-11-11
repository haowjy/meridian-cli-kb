# Per-operation cost

Launch, close, wait, show, cancel and continue concern one spawn and one chat, so their work should not grow with project history. Global scans belong to explicit global commands or bounded maintenance.

## Where cost still grows with history

- **The session journal.** Every chat appends to one `sessions.jsonl` ([session state](session-state.md#sessionsjsonl-is-one-global-journal)). Per-operation reads go through one keyed seam with a byte prefilter, so they decode only the chats they need, but each pass still reads the whole file, and a primary close makes about ten such passes. A no-op foreground launch took 3.6, 6.0 and 9.1 s on 1, 19 and 47 MB journals. Per-chat journals are the chosen fix, not yet built ([#418](https://github.com/haowjy/meridian-cli/issues/418)). Bulk readers still outside the seam: listing all session records, listing generations, and the legacy native import.
- **Session teardown reads every spawn's state** ([#612](https://github.com/haowjy/meridian-cli/issues/612)).
- **History index.** Readers take the exclusive catch-up gate even when nothing changed ([#594](https://github.com/haowjy/meridian-cli/issues/594)); a cold build holds it for the whole build, 2.4–5.7 s ([#592](https://github.com/haowjy/meridian-cli/issues/592)); indexed spawn reads materialize every row, about 2 s at 12,000 spawns ([#600](https://github.com/haowjy/meridian-cli/issues/600)).
- **Doctor and lock GC scan everything** ([#601](https://github.com/haowjy/meridian-cli/issues/601)).
- **Memory.** Runner and primary processes hold hundreds of MB; one full materialization of spawns or sessions can set a process's lifetime RSS peak, because Python keeps the arenas ([#627](https://github.com/haowjy/meridian-cli/issues/627)).
- **A fixed floor.** A no-op launch costs about 3.6 s even with tiny history, from interpreter and subprocess startup ([#595](https://github.com/haowjy/meridian-cli/issues/595)).

## Cheaper must stay as safe

Before narrowing a scan, wait or poll, list the guarantees it provided and check the faster path keeps them; a test showing the work disappeared can't show a safety refusal still works.

- A per-chat candidate index must be followed by authoritative reads: a stale or incomplete index can't authorize a lifecycle decision.
- A keyed lookup replaces a scan only if every writer of the fact updates that key, and a graph check must follow every relevant edge, not a convenient representative.
- A disposable projection needs a real rebuild path; exercise deletion with gaps in its source state.
- For ordering equivalence, test ties, backward clocks, interleaved writers and reused keys.
- A faster wake can reorder an arbitration race ([spawn outcome](spawn-outcome.md#one-writer-first-terminal-wins)).

## Measuring

Count physical reads, scans and lock acquisitions per complete command, at several history sizes, rather than timing on a shared machine: a filtered pass still reads every byte, and timing isolated seams misses work elsewhere in the path. Count filtered and unfiltered passes together, because a replacement can remove one kind of scan while multiplying another; for fan-out operations compare wall time and lock hold too. Capture call stacks for unexplained counts. Warm a copied index before measuring warm behavior and report the cold build separately; compare old and new on the same corpus, alternating runs, and report correctness beside timing.

A deadline error can mean a lock holder is slow, not that the deadline is too short. Measure the uncontended operation and find what runs under each lock before adding retries. A held-lock reproduction tests recovery but doesn't identify the holder, and pending-source labels may name bystanders rather than the lock that blocked progress. Check a cause ranking against the locking protocol: trace who waits on each candidate and what it holds while waiting. A marker that stays pending under churn can make readers wait for later writers, even when the expensive work runs before the writer sets it.
