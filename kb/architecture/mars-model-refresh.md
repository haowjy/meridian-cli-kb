# Mars model refresh

Catalog refresh and harness probes are separate concerns, controlled together by the refresh flags. `--no-refresh-models` uses cached catalog and probe data; it does not promise that all subprocesses are suppressed, because other capability or authentication checks may still run. `MARS_OFFLINE` and probe skip differ: offline can make a probe unavailable even when a stale cache exists. See the mars-agents checkout's `src/models/AGENTS.md` for current behavior.

Meridian's primary dry run requests cache-only model resolution; cache-only and normal results use separate operation cache keys so a cache-only failure cannot suppress a later refresh. See [model resolution decisions](../decisions/model-resolution.md).

