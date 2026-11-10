# Attempt facts

## One consumer per event stream

`SpawnManager` consumes the connection's event stream and fans events out to hooks and subscribers. Claude's stream permits one consumer, so any second reader would compete with the drain loop. Pi's raw `agent_end` stays observable while its decoded attempt outcome waits for `agent_settled`. Ordinary compaction and automatic runs have their own lifecycle evidence; `summarization_retry_*` and `queue_update` stay raw telemetry with no activity meaning.

## Attempt folds

Attempt facts are folded from the events one turn actually observed, not from a time window over a later transcript; native transcripts serve conversation reads separately ([native transcript reads](native-transcript-reads.md)). If a fold fails, a known harness-specific usage total survives, but a generic partial sum is discarded rather than shown as a total. Unknown usage is not zero. A crashed runner loses its in-memory fold, so recovery relies on durable report and lifecycle evidence.
