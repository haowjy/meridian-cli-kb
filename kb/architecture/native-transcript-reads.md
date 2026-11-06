# Native transcript reads and search

## One resolver, one reader

Tracked chats read only their bound native source. A missing transcript fails rather than substituting a different session or runner output. Explicit archive and restore references can select sealed snapshots; a live chat never silently falls back to one. See [native-only history](../decisions/native-only-history.md), [session identity](../decisions/native-session-identity.md), and the source checkout's `src/meridian/lib/ops/.context/CONTEXT.md`.

Archive capture selects the exact run generation and defers while a live run observes the chat. A last-observed chat may have no transcript yet; its absence is not evidence for falling back to the entry chat. Pi's normal tracked view follows its reopen lineage; an explicit file read can show raw journal order.

## Search projection

The native search index is disposable and cannot select a chat's source. It nominates candidates, which are verified against normalized native text. A freshness witness brackets parsing so an append during indexing cannot be hidden behind a stale projection. Search reports incomplete coverage when sources are missing, unsupported, changed, or not reached within the budget. See [history storage](../decisions/history-storage.md) and [native-only history](../decisions/native-only-history.md).

On a 2,037-key corpus, warm searches took 1.23–1.77 seconds and matched brute-force hit sets for seven queries; the search projection used 218 MB. The under-one-second target was not met: about 0.44 seconds was query work and the remainder largely CLI startup, tracked in [#527](https://github.com/haowjy/meridian-cli/issues/527). These are measurements, not guarantees.

