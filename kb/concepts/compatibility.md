# Compatibility between builds

Different Meridian builds share state: a long-running primary survives an upgrade, and synced configuration reaches machines on different versions. The policy itself is in meridian-cli `docs/releasing.md#compatibility-and-versioning`; this page holds its reasons.

## New builds read old data; old builds refuse new data

A deprecation lasts one minor release: long enough for machines sharing configuration to upgrade, short enough to bound compatibility code. Refusing unknown data stops a default substituted on parse failure from looking valid and later overwriting the original. Scripts consume JSON output fields, so removing one is a compatibility change too.

## Integer format versions only where another build reads

A version field nobody checks promises handling that doesn't exist, so it is deleted. Mars's `requires-meridian` check when the environment version is absent is unresolved ([mars-agents #178](https://github.com/haowjy/mars-agents/issues/178)).

## Readers learn a version one release before writers write it

An older reader rejects a field whose version it doesn't know; shipping the reader first keeps the new writer from breaking it.

## Version checks guard decisions and mutations, not blind appends

A blind append interprets no prior rows, so checking every existing row adds no safety and costs a full read per append (0.25 s at 40,000 rows). Legacy checks are scoped to the affected chat, so one unsupported row doesn't block every command.

## Rebuildable indexes

An index's file name carries every input format it depends on, forcing a rebuild when any interpretation changes. Unsupported rows are skipped and recorded, so later sessions stay reachable instead of the index freezing at the first unknown row. See [portable history](../architecture/state-system/portable-history.md#indexes).

## Idle state treats an unknown file as absent

Idle files are short-lived. Refusing one after an upgrade would disable idle handling; treating it as absent loses at most the current stretch.
