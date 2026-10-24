# Spawn Wait Barrier

`meridian spawn wait` is the **synchronization barrier** for multi-agent
orchestration. After launching background subagents, the orchestrator runs
`meridian spawn wait` to block until all pending work in the current chat
completes before producing a final answer.

---

## No-arg Wait by Chat Lineage

```bash
meridian spawn wait
```

With no positional spawn IDs, `wait` discovers the caller's active wait set and
the finished background children still owed to that caller.
Active discovery is lineage-scoped: all spawns at any nesting depth inherit the
same root chat ID.

**Discovery filters:**

| Filter | Detail |
|---|---|
| Owner chat (`owner_chat_id`, else `chat_id`) `== MERIDIAN_CHAT_ID` | Only spawns belonging to this chat tree |
| Active set | `queued`, `running`, or `finalizing`; never capped |
| Owed finished set | Oldest 32 terminal, unconsumed, unleased background direct children; excess stays owed |
| Self-exclusion | The current spawn (`MERIDIAN_SPAWN_ID`) is excluded from the wait set |
| **Descendant-scoping** | When called from a nested spawn, only descendants of the caller are included (not siblings, ancestors, or the primary session) |

If `MERIDIAN_CHAT_ID` is unset (not running inside a harness session), the
no-arg form returns an error — there is no meaningful chat scope to resolve.

Since #599 (v0.10.8), the history index applies these filters in SQL. A root
caller asks for active rows whose owner chat is `MERIDIAN_CHAT_ID`, and a nested
caller asks for its active descendant projection. The traversal keeps terminal and
archived ancestors, so active grandchildren under an archived parent are still
found. The index only selects membership: each selected spawn is re-read from its
authoritative `state.json` before wait decides it is pending. Before #599,
discovery materialized every indexed row and filtered in Python. The measured
before/after cost is recorded on the [per-operation cost bar](../architecture/per-operation-cost.md).
The SQL filter keeps discovery within that bar.

The owed set comes from the same `owed.py` definition as Claude's hidden
`spawn collect`: background direct children of every primary generation in the
chat (only the caller's own for a nested caller), minus every recorded
consumption and live lease ([what a lead is owed](child-result-delivery.md#what-a-lead-is-owed)).
Owed rows are resolved once, never enter the polling loop or the "Waiting for N
pending" table, appear under `owed_spawns` as header-only entries, and cannot
fail the wait. Rows active at resolution stay under `spawns` with their full
reports. Field-level contract: meridian-cli
`src/meridian/lib/ops/spawn/.context/CONTEXT.md` ("No-Arg Wait Scoping");
rationale: [D-no-arg-wait-owed-bounded](../decisions/child-result-delivery.md#d-no-arg-wait-owed-bounded).

An observing wait leases both classes and consumes them in its `finally`,
before the CLI prints. A wait killed between the two loses that output, and the
results stay consumed. Collect can also deliver an owed child once more if its
last check ran just before the wait leased it
([accepted duplicates and losses](child-result-delivery.md#accepted-duplicates-and-losses)).

If no pending spawns exist, the command returns immediately with a clean no-op
message. That answer requires complete history-index coverage. Discovery reads
the index for at most `QUERY_TIMEOUT` (2 s), even when the wait budget is 3000 s,
so one discovery never holds the exclusive catch-up gate for the whole wait. Since
#588 (v0.10.4), if the index is still busy, wait exits 0 with a checkpoint instead
of an error, and the orchestrator re-runs it:

```
Wait checkpoint after 0s.
Discovery still pending (history index busy); run `meridian spawn wait` again.
```

Before v0.10.4 the same contention surfaced as `History index deadline exhausted`
and exit 1 ([decision](../decisions/history-storage.md#d-history-index-readers-never-wait-on-session-writers)).
On a freshly copied or cold index, the checkpoint can appear on every call until the
first catch-up finishes.

### Descendant-Scoping in Nested Spawns

When `MERIDIAN_SPAWN_ID` is set (the wait is called from inside a spawned
agent, not from the primary session), `spawn wait` restricts the wait set to
**descendants of the caller only**. Siblings, ancestors, and the primary
session are excluded.

Wait discovery gets this set from the history index's descendant projection,
restricted to active rows (`_discover_pending_spawns` in
`lib/ops/spawn/api.py`). It then re-reads each selected row from authority.

**Why this matters:** Without descendant-scoping, a tech-lead or orchestrator
agent calling `spawn wait` would block on the primary session (which is running
in the same chat) and on any sibling spawns launched by the primary. This
produced infinite wait loops where nested spawns blocked on their own parent —
the caller's chat tree never drained because the primary session itself was in
the wait set.

**Primary session behavior is unchanged.** When `MERIDIAN_SPAWN_ID` is not
set, discovery covers the full chat tree as before — this is the primary's own
no-arg wait, which correctly waits for all work it launched.

**Explicit-ID wait is unchanged:**

```bash
meridian spawn wait p2404 p2408
```

Explicit waits still work as before, and also display the wait set before
blocking (see [Wait-Set Display](#wait-set-display)).

---

## Wait-Set Display

Before blocking, `spawn wait` prints the full wait set — both for no-arg and
explicit-ID invocations:

```
Waiting for 3 pending spawn(s) for chat c123:
  spawn  agent     model        status
  p2404  explorer  gpt-5.4      running
  p2408  reviewer  claude-opus  finalizing
  p2410  -         -            running
```

Identity comes first (spawn, agent, full model ID) so an orchestrator can
record who it is waiting on without a separate lookup; missing values render
as `-` so columns stay aligned. The table omits descriptions so the output an
orchestrator re-reads on every wait stays token-efficient. `spawn children` text
uses the same columns plus `duration`; its JSON output still carries each
child's `desc`. Checkpoint lines (see [Checkpoint Output](#checkpoint-output))
still append the description.

The wait set is **frozen at discovery time**. Spawns that become queued after
discovery are not added. This is intentional: the orchestrator controls when to
open a new barrier, not the barrier itself.

---

## Fail-Fast

```bash
meridian spawn wait --fail-fast
meridian spawn wait --fail-fast p2404 p2408
```

`--fail-fast` exits as soon as any spawn in the wait set reaches `failed` or
`timed_out`, without waiting for the remaining spawns to finish. The output
includes which spawns triggered the early exit and which are still pending.
Only blocked-on `spawns` participate. A failed child
that was already terminal at target resolution is delivery, not failure reached
while this wait blocked, and cannot trigger fail-fast or exit 1.

**Trigger statuses:** `failed` and `timed_out` only. `cancelled` does not
trigger fail-fast -- cancellation keeps existing aggregate-exit behavior where
the orchestrator handles all terminal results together.

**Snapshot consistency:** When fail-fast fires, the implementation re-reads
all pending spawns from state in one pass and derives the `pending_ids` from
that same snapshot. This prevents a spawn that goes terminal during the
re-read from appearing as both failed and pending. If the re-read shows
everything went terminal, the normal completed output is returned instead.

**Output shape:** `SpawnWaitMultiOutput` with `fail_fast=True` and
`pending_ids` listing the spawns that are still running. Text output leads
with a summary line naming the failed IDs and the pending IDs. JSON output
adds `fail_fast` and `pending_ids` fields to the wire shape.

---

## Yield Semantics

`spawn wait` uses a **yield-checkpoint** model, not a hard timeout:

- The default behavior is to **yield** after a configurable interval when
  spawns are still pending, print a continuation message, and exit with
  code 0.
- Checkpoint expiry is **not failure**. It returns control to the calling
  harness within its configured idle/compaction budget.
- A **hard timeout** (`--timeout`) causes `failed` semantics and is distinct
  from yield. When `--timeout` is not explicit, `hard_deadline = None` and
  only the checkpoint governs.

### Checkpoint Output

When the yield interval expires and spawns are still pending:

```
Wait checkpoint after 4m. Still pending:
  p2408  running  Review design
    waiting for descendants: p2411, p2413

Run `meridian spawn wait` again to continue.
```

Since #708 (v0.11.3), a checkpoint lists active descendants beneath each
blocked spawn. This makes recursive completion blockers visible without
changing the frozen wait set or the caller's checkpoint duration. JSON output
exposes the same mapping as `checkpoint_descendants`.

For explicit-ID waits, the continuation command includes remaining IDs:

```
Run `meridian spawn wait p2408` again to continue.
```

The orchestrator re-runs `meridian spawn wait` to resume blocking. There is
no state to restore — rediscovery happens fresh on each invocation.

---

## Harness-Aware Yield Defaults

The default yield interval belongs to the **caller's** harness, not the
harnesses of the spawns being waited on. It bounds how long that caller remains
inside one wait before it can checkpoint again. It is not a promise about
provider prompt-cache lifetime.

A resident child's completion deadline inherits that same caller setting unless
explicitly overridden, but the timers are independent: wait expiry yields;
completion deadline expiry can time out the child. See
[drain plans](../architecture/drain-plans.md).

Harness detection reads `os.getenv("_MERIDIAN_HARNESS")`, which
`bind_launch_context()` sets in every launched process's environment
(see [Launch System](../architecture/launch-system.md#_meridian_harness-child-env-injection)).

| Harness | Default yield interval |
|---|---|
| all harnesses (unified) | 3000 seconds (50 minutes) |
| minimum clamp | 30 seconds |

All harnesses default to 3000 seconds. If a spawn record lacks harness
metadata, the same unified value applies.

Config keys: `[spawn].default_wait_yield_seconds`, `[spawn].min_wait_yield_seconds`,
`[harness.claude].wait_yield_seconds`, `[harness.codex].wait_yield_seconds`,
`[harness.opencode].wait_yield_seconds`.

Override the operational checkpoint interval with
`--yield-after-secs` per invocation, or `MERIDIAN_WAIT_YIELD_AFTER_SECONDS`
environment variable for a global override.

### Per-Invocation Override

```bash
meridian spawn wait --yield-after-secs 120
```

`--yield-after-secs` overrides the harness-derived default for a single
invocation. The flag name deliberately avoids "timeout" to signal that this is
a continuation point, not a failure path.

---

## Background Spawn Obligation

When a background spawn is submitted (`--bg`), the CLI output includes an
explicit MUST-run obligation:

```
Background spawn submitted.
Spawn id: p2404

After spawning all subagents, you MUST run:

  meridian spawn wait

This waits for all pending spawns for this chat.
```

The no-arg form collects all background spawns in the current chat tree — no
per-spawn ID tracking required.

A lead that ignores the obligation and goes idle is not stuck on every harness:
interactive Claude primaries and Pi sessions get owed results delivered as a new turn. Each wait
records what it returned as consumed in `observed-spawns.json`; owed discovery
also honors Pi delivery receipts and explicit dismissals, so another route does
not replay it. See
[Child result delivery](child-result-delivery.md).

---

## Behavioral Notes

### `--yield-after-secs`, not `--timeout-secs`

The flag was named `--yield-after-secs` rather than `--timeout-secs` because
"timeout" implies failure. The yield is a continuation point
that exits 0 with guidance. "Timeout" is reserved for hard failure semantics
(the existing `--timeout` flag).

### Hard timeout disabled in checkpoint mode

When `--timeout` is not explicitly provided, `hard_deadline = None` and only
the checkpoint governs. This prevents config-level `wait_timeout_minutes` from
silently preempting checkpoint behavior. The config timeout is a hard cap for
explicit timeout invocations; it must not shorten normal checkpoint-mode waits.

### Wait set frozen at discovery time

The wait set is collected once at the start of the `wait` invocation and not
updated as spawns are launched afterward. This is by design: the orchestrator
controls the boundary between "launching phase" and "barrier phase." A barrier
that auto-expands is unpredictable and can delay completion indefinitely.

### Vanished spawns in no-arg discovery

If a spawn is discovered but disappears from the store before the first poll
(e.g., a race where it completes between discovery and polling), it is treated
as **resolved**, not as an error. This applies only to no-arg discovery.
Explicit-ID waits retain strict behavior — a vanished explicit spawn is an
error.

---

## Related Pages

- [Spawn Lifecycle](spawn-lifecycle.md) — status machine, reaper, and
  projection authority
- [State Model](state-model.md) — JSONL event stores and crash-only design
- [Child Result Delivery](child-result-delivery.md) — how results reach a lead that didn't wait, and the consumption record waits write
- `../operations/configuration-guide.md` — TOML config and env var reference
- [Spawn Output Contract](spawn-output-contract.md) — what `spawn wait` prints when a single spawn completes; report-first default, transcript pointer, progressive disclosure
- [Per-operation cost](../architecture/per-operation-cost.md) — discovery's
  constant-time bar and measured acceptance evidence
- [Session state](../architecture/state-system/session-state.md) — session
  authority and the history-index source behind discovery
- [Contention and cost diagnosis](../lessons/contention-diagnosis.md) — why wait
  reports a checkpoint instead of hiding index contention
- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md) — provider/authentication cache policy, separate from the operational wait checkpoint
