# Session Browse

The picker shows recognizable session details and opens transcripts on request. The interface is documented in meridian-cli `docs/history.md`; its framework choice is in [session-browse TUI](tui-framework.md).

## Recognition rows {#d-browse-recognition-row}

Idle time and a cache-expiry warning help estimate the cost of reentry. Live and stopped sessions sort together; reentry forks a live session rather than attaching twice. A permanently split preview was rejected because ordinary terminal heights clipped most transcripts.

## Transcript on demand {#d-browse-transcript-on-demand}

Highlighting a row does not read its transcript. An on-demand full-screen view avoids both clipped previews and a separate preview cache that could become stale or identify the wrong session.

## Cache staleness {#d-browse-staleness-from-idle-state}

Compute staleness from existing idle and stop facts rather than adding a journal write on every turn. A running session with no idle evidence has an unknown deadline, not a guessed warning.

## Search before limit {#d-browse-query-before-limit}

Filtering before the row limit prevents a matching session beyond the first page from appearing absent. One query defines matching for both the count and the visible rows.

## Lineage {#d-browse-lineage-view}

The lineage view shows primary-session `--fork` and `--from` ancestry, not subagent spawn lineage. See [conversation identity](state.md#d-lineage-conversation-identity).

## Start from a row {#d-browse-start-from}

Starting from a session inherits its agent, last-used model and work. Running sessions qualify because this reads recorded context rather than attaching to the live session.

## Python first {#d-browse-python-first}

The picker remained in Python because profiling found startup and state reads dominated latency, not drawing. A separate Rust frontend would duplicate the state model or pay for the same Python reads. The measured first paint after the redesign was 0.48–0.60 s, above the 200 ms target.
