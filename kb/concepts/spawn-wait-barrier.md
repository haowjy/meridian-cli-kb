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

With no positional spawn IDs, `wait` discovers every **pending** spawn that
shares the current root chat, identified by `MERIDIAN_CHAT_ID`. Discovery is
lineage-scoped: all spawns at any nesting depth inherit the same root chat ID.

**Discovery filters:**

| Filter | Detail |
|---|---|
| `chat_id == MERIDIAN_CHAT_ID` | Only spawns belonging to this chat tree |
| Active status only | `queued`, `running`, or `finalizing` |
| Self-exclusion | The current spawn (`MERIDIAN_SPAWN_ID`) is excluded from the wait set |
| **Descendant-scoping** | When called from a nested spawn, only descendants of the caller are included (not siblings, ancestors, or the primary session) |

If `MERIDIAN_CHAT_ID` is unset (not running inside a harness session), the
no-arg form returns an error — there is no meaningful chat scope to resolve.

If no pending spawns exist, the command returns immediately with a clean no-op
message.

### Descendant-Scoping in Nested Spawns

When `MERIDIAN_SPAWN_ID` is set (the wait is called from inside a spawned
agent, not from the primary session), `spawn wait` restricts the wait set to
**descendants of the caller only**. Siblings, ancestors, and the primary
session are excluded.

This is implemented via BFS from the caller's spawn ID through the
`_collect_descendants()` tree walk in `ops/spawn/api.py`. The scope is then
intersected with the active-status filter.

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
Waiting for 3 pending spawns for chat c123:
  p2404  running     Design mermaid check style warnings
  p2408  finalizing  Review design package
  p2410  running     Probe CLI behavior
```

The wait set is **frozen at discovery time**. Spawns that become queued after
discovery are not added. This is intentional: the orchestrator controls when to
open a new barrier, not the barrier itself.

---

## Yield Semantics

`spawn wait` uses a **yield-checkpoint** model, not a hard timeout:

- The default behavior is to **yield** after a configurable interval when
  spawns are still pending, print a continuation message, and exit with
  code 0.
- Checkpoint expiry is **not failure**. It is a cache-preserving pause designed
  to let the calling harness issue another tool call before session context
  expires.
- A **hard timeout** (`--timeout`) causes `failed` semantics and is distinct
  from yield. When `--timeout` is not explicit, `hard_deadline = None` and
  only the checkpoint governs.

### Checkpoint Output

When the yield interval expires and spawns are still pending:

```
Wait checkpoint after 4m. Still pending:
  p2408  running  Review design

Run `meridian spawn wait` again to continue.
```

For explicit-ID waits, the continuation command includes remaining IDs:

```
Run `meridian spawn wait p2408` again to continue.
```

The orchestrator re-runs `meridian spawn wait` to resume blocking. There is
no state to restore — rediscovery happens fresh on each invocation.

---

## Harness-Aware Yield Defaults

The default yield interval is **harness-aware**. The intent is conservative
safe-yield assumptions chosen for cache/session preservation, not TTL modeling.

The interval is determined by the **caller's** harness — the harness the
orchestrator itself is running inside — not by the harnesses of the spawns
being waited on. The yield protects the orchestrator's own prompt-cache
context window, so the orchestrator's session TTL is the right signal.

Harness detection reads `os.getenv("MERIDIAN_HARNESS")`, which
`build_launch_context()` sets in every spawned process's environment
(see [architecture/launch-system.md](../architecture/launch-system.md) —
MERIDIAN_HARNESS child env injection).

| Harness | Default yield interval |
|---|---|
| all harnesses (unified) | 3000 seconds (50 minutes) |
| minimum clamp | 30 seconds |

All harnesses default to 3000 seconds because extended prompt-cache TTLs
are available for Claude Code (`ENABLE_PROMPT_CACHING_1H=1`), Codex, and
OpenCode. If a spawn record lacks harness metadata, the same unified value
applies.

Config keys: `[spawn].default_wait_yield_seconds`, `[spawn].min_wait_yield_seconds`,
`[harness.claude].wait_yield_seconds`, `[harness.codex].wait_yield_seconds`,
`[harness.opencode].wait_yield_seconds`.

Users who cannot extend harness cache TTL should lower the yield interval:
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

---

## Behavioral Notes

### `--yield-after-secs`, not `--timeout-secs`

The flag was named `--yield-after-secs` rather than `--timeout-secs` because
"timeout" implies failure. The yield is a cache-preserving continuation point
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
- `../operations/configuration-guide.md` — TOML config and env var reference
- [Spawn Output Contract](spawn-output-contract.md) — what `spawn wait` prints when a single spawn completes; report-first default, transcript pointer, progressive disclosure
