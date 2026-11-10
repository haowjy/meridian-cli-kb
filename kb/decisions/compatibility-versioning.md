# Compatibility and Format Versioning

The compatibility policy is in meridian-cli `docs/releasing.md#compatibility-and-versioning`. Different builds can share state when a long-running primary survives an upgrade or a repository syncs across machines.

## One-minor deprecation; new builds read old data, old builds refuse new data

One minor release gives machines sharing configuration time to upgrade while bounding compatibility code. Refusing unknown data prevents a default substituted on parse failure from looking valid and later overwriting the original. Scripts also consume JSON output fields; treat their removal as a compatibility change.

## Integer format versions where another build reads; delete the rest

A version field nobody checks promises compatibility handling that does not exist. See [Mars #178](https://github.com/haowjy/mars-agents/issues/178) for the unresolved `requires-meridian` check when the environment version is absent.

## Readers learn a version one release before writers write it

Older readers can reject a new field before they understand its version; staging prevents the new writer from breaking them.

## Version checks guard decisions and mutations, not blind appends

Blind appends interpret no prior rows, so checking all existing rows adds no safety. A full read cost 0.25 seconds per append at 40,000 rows and 0.45 seconds at 73,000. Scope legacy import checks to the affected chat so one unsupported row does not block every command. Rows naming no chat or native session remain invisible to scoped decisions.

## A rebuildable index's file name carries every input format

Including independently versioned inputs forces a rebuild when their interpretation changes. Skipping and recording unsupported rows preserves access to later sessions instead of freezing the index at the first unknown row.

## Idle state warns and behaves as absent

Idle files are short-lived; refusing an unknown file after an upgrade would disable idle handling, while treating it as absent loses at most the current stretch.
