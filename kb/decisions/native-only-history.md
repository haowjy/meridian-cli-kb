# Conversation History Is Native-Only

A chat's conversation comes from the harness transcript selected by its recorded native binding. A runner stream can mix two native conversations after an in-TUI switch and must not act as a fallback authority. On one measured installation the duplicate runner histories occupied 27.1 GB across 4,392 files. See [native transcript reads](../architecture/native-transcript-reads.md) and [native session identity](native-session-identity.md).

## Old runner history

Old runner history is not read, even when it is the only remaining conversation. A measured 155 chats, about 850 MB, had only that copy. Keeping a decoder or fallback would preserve a second, unverifiable read path. Provable old bindings can be recovered by [legacy native import](legacy-native-import.md).

### Related retention decisions (same day)

A conversation becomes unavailable if the native transcript disappears and no archive contains a snapshot. Harness retention and regular archival therefore matter; see the [history guide](https://github.com/haowjy/meridian-cli/blob/main/docs/history.md).

## Search: a native-keyed trigram projection, verified exactly

The search index nominates candidates; native text verifies matches. It is disposable and cannot select a chat's source. In one 2,149-chat corpus, parsing at query time took 12.7 seconds, while a contentless trigram projection with exact verification found all 419,253 known matches at about 200 MiB. SQLite token search and raw trigram text missed substring or Unicode cases. See [native transcript reads](../architecture/native-transcript-reads.md#search-projection).

The metadata index shares the authoritative session fold; a separate fold can lose a native key on resume or accept a conflicting generation. Attempt facts come from events the run actually observed, not a time window over a later transcript; incomplete generic usage must not erase a harness's authoritative total. See [attempt facts and delivery](../architecture/attempt-facts-and-delivery.md).

### Dogfood rows migrate once, not on read

Early native-history rows migrate through doctor or background repair. Read-time rewriting could run under the non-reentrant spawn lock or overwrite a concurrent mutation. See [health checks](../operations/health-checks.md#dogfood-row-migration).

### The prune rule

Pruning retired runner history is explicit and dry-run first. It requires an exactly resolving native source and revalidates before deletion; live or unresolved runs are skipped. See [session archive pruning](../operations/session-archive-pruning.md#runner-history-prune).
