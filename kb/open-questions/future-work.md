# Future work and known limitations

Entries without issue links are untracked and need verification before work starts.

## Managed primary

[Primary message routes](../decisions/primary-inject.md#session-moves) cannot reliably follow TUI session moves. OpenCode can send to the old session before the human types in the new one; a Codex subagent resumed without `thread/started` may be mistaken for a move (inferred, not reproduced). OpenCode V1 move detection is untested. An old Pi listener could unlink a newer listener's socket if the old process outlives its launcher. See [#916](https://github.com/haowjy/meridian-cli/issues/916).

## State & spawn

- [#537](https://github.com/haowjy/meridian-cli/issues/537): restore metadata hashes depend on reader models. Hash the stored event JSON instead.
- [Per-operation cost](../architecture/per-operation-cost.md) tracks work whose cost grows with project history.
- [#701](https://github.com/haowjy/meridian-cli/issues/701): measure whether `--fork` preserves prompt cache before deciding whether it and `--fork-fresh` should merge.
- Nested `doctor --kill-orphans` skips the reaper but its warning can imply reconciliation ran. Untracked.
- The legacy repo-local migration helper is a registry stub and is not run at startup; decide whether to support it or remove it. Untracked.
- Spawn CLI and streaming serve have separate preparation paths; [#700](https://github.com/haowjy/meridian-cli/issues/700) tracks repeated background composition.

## Streaming and completion

- Background-work visibility differs by harness; probe it before removing a confirmation ask. See the source checkout's streaming completion contract. Untracked.
- `tests/integration/launch/test_spawn_manager_lifecycle.py` seeds private `_sessions`; a manager-owned setup could avoid that coupling. Untracked.
- [#438](https://github.com/haowjy/meridian-cli/issues/438): terminal error normalization loses provenance; Pi precedence recognizes a string prefix instead of a typed origin.
- [#439](https://github.com/haowjy/meridian-cli/issues/439): cleanup telemetry does not distinguish connection teardown from tracked-work cleanup.

## Routing and model resolution

Mars agent overlays lack an agent-specific timeout. Routing diagnostics may omit resolution layers, available choices and a `meridian list` mode filter; plain `MERIDIAN_HARNESS` is not a harness override. These are untracked and need verification against current code.

## Pi result delivery limits {#pi-result-delivery-limits}

A message queued but never admitted may keep its IDs reserved and pause unrelated automatic publication until a native lifecycle callback, reload or restart. A real reproduction is still needed. Fresh tool-boundary discovery scans all spawn rows; revisit if measured latency warrants a safe incremental index. Backlogs larger than a packet need multiple native opportunities. See the source checkout's `src/meridian/pi_runtime/.context/delivery-contract.md` and [#561](https://github.com/haowjy/meridian-cli/issues/561).

Managed Bash duplicates foreground attachment bookkeeping; consolidate only if doing so preserves model and human output, physical stop, publication and reload behavior. Its post-shell process-group polling has exploratory backoff measurements but lacks full-runtime and macOS qualification. Both are untracked.

## Telemetry and code structure

Telemetry lifecycle observers and bootstrap cross process boundaries, and event readers duplicate truncation-tolerant JSONL parsing. Session-reference handling and bootstrap carriers overlap; `CodexConnection` spans multiple concerns. Establish the state read and repair contract before collapsing `session_target.py`. Untracked.

## Jupyter Workbench

Verify these untracked follow-ups against current source: repeated session-id and manifest handling; raw mutable DTO fields; non-atomic notebook writes; replay coupled to execution mechanics; and replay's inability to reconstruct live browser visualization. See [architecture](../ecosystem/jupyter-workbench/architecture.md) and [runtime model](../ecosystem/jupyter-workbench/runtime-model.md).

## MicroCT analysis

Verify these untracked follow-ups against current source: overlapping PCA implementations in `orientation_tools.py` and `processing/orientation.py`; potentially shared slice extraction if a third consumer appears; and the missing stateful prompt-test harness for `slice-examination-loop`.
