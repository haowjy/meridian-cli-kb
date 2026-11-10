# Native transcript reads and search

A chat's conversation comes only from the harness transcript its recorded [native binding](native-session-binding.md) selects. Contract: the source checkout's `src/meridian/lib/ops/.context/CONTEXT.md`; user guide: `docs/history.md`.

## Native transcripts are the only source

A missing transcript fails the read rather than substituting another session or runner output. The runner's own stream can mix two native conversations after an in-TUI switch, so it can't serve as a fallback; on one installation those duplicate runner histories also took 27.1 GB. The transcript path is resolved from the recorded store and ID each time, not stored, because a transcript can be re-seeded into another store.

Old runner history is not read, even where it is the only copy left (155 chats on one measured installation). A decoder kept for it would be a second read path nobody can verify. Old bindings that can be proven are recovered by [legacy native import](legacy-native-import.md); pruning the retired runner history is explicit and dry-run first, needs an exactly resolving native source, revalidates before deleting, and skips live or unresolved runs ([session archive pruning](../operations/session-archive-pruning.md)).

A conversation becomes unavailable when its native transcript disappears and no archive holds a snapshot, so harness retention settings and regular archiving matter. Explicit archive and restore references can select sealed snapshots; a live chat never silently falls back to one ([portable history](state-system/portable-history.md)).

A last-observed chat may have no transcript yet; that absence is not a reason to fall back to the entry chat. Pi's tracked view follows its reopen lineage, while an explicit file read can show raw journal order ([Pi sessions](pi-native-sessions.md#journal-topology-and-readback)).

## Search: an index nominates, native text verifies

The search index is a disposable trigram projection keyed by native source. It nominates candidates, each verified against normalized native text, and it can never select a chat's source. Parsing every transcript at query time took 12.7 s on a 2,149-chat corpus; SQLite token search and storing raw trigram text both missed substring or Unicode matches, while a contentless trigram projection with exact verification found all 419,253 known matches in about 200 MiB.

A freshness witness brackets parsing, so an append during indexing can't hide behind a stale projection. Search reports incomplete coverage when sources are missing, unsupported, changed or not reached within its budget. Warm searches take 1–2 s, short of the sub-second target; most of the rest is CLI startup ([#527](https://github.com/haowjy/meridian-cli/issues/527)).

The metadata index shares the authoritative session fold: a separate fold could lose a native key on resume or accept a conflicting generation.

## Early rows migrate once, not on read

Early native-history rows migrate through doctor or background repair. Rewriting at read time could run under the non-reentrant spawn lock or overwrite a concurrent change ([health checks](../operations/health-checks.md)).
