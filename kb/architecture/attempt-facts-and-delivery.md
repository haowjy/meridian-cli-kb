# Attempt facts and delivery

## The emit path

Claude's event stream permits one consumer, so the drain loop normalizes events once and fans them out to hooks and subscribers. Pi's raw `agent_end` remains observable while its decoded attempt outcome waits for `agent_settled`. Ordinary compaction and automatic runs have separate lifecycle evidence; `summarization_retry_*` and `queue_update` remain raw telemetry without activity semantics. See [connection seams](../decisions/connection-seams.md) and the source checkout's streaming code.

## Attempt folds

Attempt facts are folded from one turn's live events; native transcripts serve conversation reads separately. If a fold fails, a known specific usage total survives but a generic partial sum is discarded rather than presented as a total. Unknown usage is not zero. A crashed runner loses its in-memory fold, so recovery relies on durable report and lifecycle evidence. See [native-only history](../decisions/native-only-history.md).
