# Spawn lifecycle

A spawn is one delegated execution. It keeps its ID across continuations, each of which is a new internal launch ([spawn identity and launches](../architecture/state-system/spawn-launches.md)). Its terminal status describes how execution ended, not whether the agent's work is correct. Contract: the source checkout's [`src/meridian/lib/ops/spawn/AGENTS.md`](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/ops/spawn/AGENTS.md).

The stages and where their reasons live:

1. **Prepare, then reserve.** A setup error leaves no row.
2. **Run under the launch lock.** The lock is held from publication through runner adoption, so a launcher that dies before handing it off leaves a dead launch readers can derive, not a stranded row ([derived status](../architecture/state-system/derived-status.md)).
3. **Hold.** A finished turn is a candidate; the [completion hold](../architecture/completion-drain.md) decides when it publishes, and a close cancels remaining work.
4. **Finalize once.** One writer publishes the terminal outcome ([spawn finalization](../architecture/spawn-finalization.md)); the result is then owed to the parent ([child result delivery](child-result-delivery.md)).

Nested `cancel-all` cancels only the caller's descendants, so a child's cleanup can't cancel its siblings or the primary's other work; `--include-others` widens it to the chat.

What a finished spawn prints: the report leads, because it is the result; a transcript pointer follows so a reader can check the source conversation when the report is incomplete. `--metadata` adds accounting without hiding the report, and `--verbose` adds diagnostics. JSON defaults are curated per command to keep routine coordination small, and scripts treat those fields as a contract ([compatibility](compatibility.md)). Formats: `docs/commands.md`.
