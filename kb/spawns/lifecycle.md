# Spawn lifecycle

A spawn is one delegated execution. It keeps its ID across continuations, each of which is a new internal launch ([spawn identity and launches](../state/spawn-launches.md)). Its terminal status describes how execution ended, not whether the agent's work is correct. Contract: the source checkout's [`src/meridian/lib/ops/spawn/AGENTS.md`](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/ops/spawn/AGENTS.md).

The stages and where their reasons live:

1. **Prepare, then reserve.** A setup error leaves no row.
2. **Run under the launch lock.** The lock is held from publication through runner adoption, so a launcher that dies before handing it off leaves a dead launch readers can derive, not a stranded row ([spawn outcome](../state/spawn-outcome.md)).
3. **Hold.** A finished turn is a candidate; the [completion hold](completion-hold.md) decides when it publishes, and a close cancels remaining work.
4. **Finalize once.** One writer publishes the terminal outcome ([spawn outcome](../state/spawn-outcome.md)); the result is then owed to the parent ([results and waiting](results-and-waiting.md)).

Nested `cancel-all` cancels only the caller's descendants, so a child's cleanup can't cancel its siblings or the primary's other work; `--include-others` widens it to the chat.

A goal belongs to the spawn, not to the work item and not only to the prompt text: spawns in one work item can have different goals, and consumers shouldn't have to parse a prompt to find one. A goal doesn't select model, harness or permissions. Its text is trimmed without template substitution, so persisted and delivered text agree.

What a finished spawn prints: the report leads, because it is the result; a transcript pointer follows so a reader can check the source conversation when the report is incomplete. `--metadata` adds accounting without hiding the report, and `--verbose` adds diagnostics. JSON defaults are curated per command to keep routine coordination small, and scripts treat those fields as a contract ([compatibility](../state/compatibility.md)). Formats: `docs/commands.md`.
