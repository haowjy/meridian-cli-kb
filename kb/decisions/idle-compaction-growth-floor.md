# Decision: Idle Compaction Only When It Pays

## D-idle-compaction-growth-floor — Compact on growth past a per-harness break-even

**Status:** decided by the user (2026-10-14, "2 yes do it"; review cuts
approved after `spawn:p7956`). Built on local branch
`feat/idle-compaction-growth-floor` (head `30cd30657`), which stacks the
unmerged PR #578 and #583 lanes. Unpushed, no PR, unreleased. **`main` and every
release through v0.10.2 still apply the old floor:** guard 13 skips only when a
*known* total context is under `idle.min_compact_tokens` (40,000), and an
unknown context compacts. Everything below describes the branch.

Provenance: `work:idle-compaction-floor` (design `design/compaction-floor.md`,
approved cuts `DIVERGENCE/review-cuts.md`, measurements in `evidence/`).
Measurements `spawn:p7919`, cache probe `spawn:p7933`, design `spawn:p7934`
reviewed by `spawn:p7942`, implementation `spawn:p7943`, strict review
`spawn:p7956`, fixes `spawn:p7958`. The cost model that started it is
[Idle Compaction Break-Even](../research/idle-compaction-break-even.md).

### Problem

Idle compaction exists to make a cold return cheap
([Idle Notifications](idle-notifications.md)). The old guard compared total
context with a fixed 40k floor, and it failed in three ways:

- A Meridian Claude primary starts at about 39k tokens, so the floor blocked
  almost nothing. A primary that had just been compacted (about 44–52k) qualified
  again in every stretch.
- Codex and OpenCode reported no context size. Unknown context failed open, so
  the floor never applied to them.
- Measured compaction costs showed that some routes never pay at all, whatever
  the size: the compaction request reads its input cold and the return is
  billed at plain input price.

### Decision

At the compact deadline, compact only when the expected saving on return beats
what compacting costs now. All costs are in base-input-token units:

```
compact  ⇔  r·w·(C − C_after)  >  c·C + u + o·S
C_after  ≈  B + Δ        Δ = 0 if B was first observed after Meridian's own compaction, else R
need     =  (r·w·Δ + c·B + u + o·S) / (r·w − c)      growth G = C − B must reach need
need     =  never                                     if r·w ≤ c
```

C is context now, B the baseline, R the compaction residue, w the cold re-read
multiplier on return, c and u the compaction request's warm-read multiplier and
flat uncached input, o and S the output multiplier and summary size, and r the
probability the user returns. Growth is an integer and the inequality is strict,
so the code uses `need = floor(x) + 1` and checks `growth >= need`.
`lib/idle/worth.py` owns the formula, and guard 13 calls it.

**Growth, not total.** The fixed prefix is paid either way, so only history
past the baseline can be saved.

**Defaults at r = 0.5:**

| Route | Costs in code | need (B from start or a shrink / B after a Meridian compaction) | Default |
|---|---|---:|---|
| Claude, 1 h TTL | w 2, c 0.1, u 20,000, o 5, S 4,000, R 13,000 | ≈63k / ≈50k | compacts when grown enough |
| Claude, any other TTL | w 1.25, rest as above | ≈99k / ≈86k | a 5 min TTL has no compact stage anyway |
| Codex | w 1, c 0.114, u 0, o 8, S 4,500, R 3,500 (assumes the OpenAI/ChatGPT route) | ≈103k / ≈100k | compacts only after large growth (o·S dominates) |
| Pi (every route), OpenCode 2.x `openai` | none: the port is absent | never | `skipped:not-worth-it`, push and warn unchanged |

The design records the measured sources for each constant (§2). Claude c and u
are fit to two Haiku probe runs that matched `0.1·C + 20k` exactly, and a
realistic primary (P8) had 19.3k uncached input, under the 20k assumed. Pi
summarizes with its own system prompt and `cacheRetention: "none"`. OpenCode's
compaction calls read no cache (median `cache.read` 0, n = 109). On those routes
c = 1 and w ≤ 2, so `r·w ≤ c` at r = 0.5.

**Never-paying routes keep the stage.** The compact stage stays on the timeline
and the guard skips it. Dropping it from `timeline.schedule` would put a second
copy of the rule outside the ordered guards and lose the persisted evaluation
that explains the skip.

**Unknown now fails closed.** An unknown context gives `skipped:context-unknown`,
and a missing baseline gives `skipped:baseline-unknown`. Like every other
non-transient compact skip, these last for the rest of the stretch. This
replaces the old D6 rule, "unknown context fails open"
([D-idle-stretch-at-most-once](idle-notifications.md#d-idle-stretch-at-most-once--each-stage-fires-at-most-once-per-stretch)).

### Cuts the user approved after review

The design's state and cost-port shape were narrowed before shipping (review
`spawn:p7956`, applied in `spawn:p7958`):

- **r is fixed at 0.5**: `RETURN_PROBABILITY` in `worth.py`. It is not config,
  and no caller passes it. The unused `return_probability` and `r=` parameters
  were deleted. `CompactEval` still records `r`, so old evaluations stay
  readable after it changes.
- **Only Claude and Codex register `compact_costs`.** Pi and OpenCode costs
  gave exactly the same verdict as having no costs at r = 0.5, so they were
  deleted (YAGNI). Their measured constants stay in design §2 for when r moves.
  At r = 1, Pi `anthropic` long retention would pay after about 70k of growth.
- **No persisted provider.** Codex never receives a provider (its notify has
  none), so its provider guard was dead code. It now assumes OpenAI. A Codex
  session on another `model_provider` gets OpenAI costs, so use the override
  there. TTL detection still gets the provider from the arm arguments.
- **Baseline is a frozen `ContextBaseline` under `IdleState.context`**
  (`context_tokens`, `baseline_tokens`, `baseline_source`). Two pure methods,
  `observe(x)` and `after_compaction()`, replace a flat set of fields that
  every arm branch had to copy by hand.
- **`compact_eval` is scoped to one stretch.** It survives re-anchors and
  resets when a stretch opens, because it explains that stretch's
  `done.compact`. `CompactEval` has no stretch field.

### Baseline lifecycle

| Event | Effect on `IdleState.context` |
|---|---|
| arm with context X > 0 (new stretch, re-anchor, `already-compacted`, duplicate turn) | B pending → B = X, source kept. X < last context (shrink: native compaction, user `/compact`, rewind) → B = X, source `observed`. Then context = X. |
| arm absorbed by `compact-window` or `expected-compaction-turn` | no change |
| **first arm after a closed stretch whose `done.compact` is `claimed` or `ok`** | **no change** |
| arm with no context, or 0 | no change |
| `done compact ok` | B pending, source `compacted` (the next real observation sets B with Δ = 0) |
| `done failed` / `vetoed`, return, `close_on_launch` (#578), new stretch | no change: the baseline outlives stretches |

**Why the closed-stretch skip exists (review S1).** A return or reopen during
a Claude compaction clears the compaction window and the expected-turn flag.
The compaction's own turn end then arrives as an ordinary arm on a closed
stretch. If Claude reports the compacted `postTokens` (about 1.2k) at that
moment, B collapses and the next real turn looks like 50k of growth, so
Meridian compacts again. Spuriously small observations are the one way the
rule over-compacts. Codex (`input_tokens == 0`, `compacted`), OpenCode (the
compaction message) and Pi (`null`) already report nothing there. Skipping
that arm costs one observation, and the next turn sets B. Service tests cover
both orders: `done ok` arriving before or after that turn end. **Not
confirmed:** whether native Claude actually emits `turn.complete` for a
compaction after a `prompt.submit`. The installed-wheel probe drove the CLI with
injected arms, so it shows the core transition, not Claude's event order.

**Why `done ok` resets instead of recording a size.** Claude's
`compact()` returns `tokensAfter`, which excludes the system prompt and tools.
Pi reports `null` until the next response. The first real turn end is the first
honest observation on every harness.

**Failure direction.** A missing, corrupt or garbage-collected state file
restarts B at the current context. Growth is then under-counted, which means
under-compacting, never over-compacting.

### Context facts: read at arm and again at fire

Each harness uses one definition of main-thread context at both points.
Growth is a difference, so only consistency matters.

| Harness | Source |
|---|---|
| Claude mod | `$.session.usage().context.tokens`, passed as `--context-tokens` on arm and fire |
| Pi extension | `ctx.getContextUsage()?.tokens`, passed on arm and fire, omitted while `null` |
| Codex | rollout tail, `token_count.info.last_token_usage.total_tokens` (`codex_rollout.latest_context_tokens`). Read at notify (`NativeIdleHook.observe` now gets `env` to find the rollout) and in sensor `facts()`. `None` at a `compacted` record or `input_tokens == 0` |
| OpenCode 2.x | newest assistant message in `/api/session/{id}/message`: `tokens.input + cache.read + cache.write`. Read at turn end and again at fire. A compaction message (`mode == "compaction"` or `summary`) stops the scan **before** the zero-usage check, so a compaction still in progress returns `None` instead of the pre-compaction message |

**Both reads stay (rejected: fire reads persisted state only).** An arm without
context deliberately leaves `context_tokens` unchanged, so a later shrink is
still visible. After a turn in which the harness compacted natively, state still
holds the old, larger size. The fire-time re-read is what turns that into
`context-unknown`. A fire that trusted state would compact a session that was
just compacted. Fixing that without the second read needs a "latest observation
unknown" flag, which is not simpler. The re-read also corrects Codex's late
`token_count` flush.

### Config and diagnostics

- `idle.compact_min_growth_tokens`, overridable per harness under
  `[harness.<h>.idle]`, replaces `idle.min_compact_tokens`. The env vars are
  `MERIDIAN_IDLE_COMPACT_MIN_GROWTH_TOKENS` and
  `MERIDIAN_HARNESS_IDLE_COMPACT_MIN_GROWTH_TOKENS_<H>`. Unset means break-even.
  An integer replaces `need` (`basis: configured`), ignores the costs, and so
  re-enables Pi and OpenCode. It still needs a known context and baseline.
  `0` compacts on any known growth.
- Rejected: an `r` setting, whose effect the user can't see (status shows
  `need`, which is what the override replaces); and `compact = "auto"`, which
  can't tune a threshold and would change an existing key's type.
- `meridian idle status --json` shows `context` and `compact_eval` (context,
  baseline, growth, need, `basis` of `break-even`/`configured`/`never`, `r`,
  costs, verdict). The table shows `skipped:not-worth-it (19k/50k)`. The `idle
  fire` wire adds `detail`, for example `growth 19k < need 50k (break-even)`.
- Upgrading needs no migration. Released rows pick up a baseline on their next
  trustworthy turn. Rows from the branch's earlier flat-field shape fail
  validation and are recreated. v0.10.0 can't read the new rows (`docs/upgrading.md`).

### Verification (branch)

- Live probes P1–P8 ran against the pre-cut implementation (`spawn:p7943`,
  `evidence/floor-live.md`). P1 matched the persisted context to each harness's
  native usage. P5 used OpenCode's free provider because the isolated `openai`
  route was unavailable. The parser is the same.
- After the cuts: a focused gate of 146 tests passed, ruff passed, and pyright
  reported 0 errors and 0 warnings. Installed-wheel scratch CLI checks covered
  both completion orders and the cost ports. The full gate was still running
  when this was written.

### Revisit when

- **r moves.** It is a judgement: 0.5 against a measured 0.27 that is biased
  low, because state keeps one stretch per file. Claude's need roughly doubles
  across the plausible range. Measuring r conditioned on reaching the deadline
  needs a per-stretch outcome record, which is deferred. If r rises, restore
  the Pi and OpenCode ports from design §2.
- **Claude's uncached input passes 20k** on real primaries: each +15k of u adds
  about 17k of need.
- **Native Claude ordering** of a compaction turn end after a return is
  observed live.
- **Codex is routed to a non-OpenAI provider** in practice.

## Related

- [Idle Notifications](idle-notifications.md): the schedule, guard order and
  stretch rules this changes.
- [Idle Adapter Architecture](../architecture/idle-adapters.md): the current
  adapter mechanism.
- [Idle Compaction Break-Even](../research/idle-compaction-break-even.md): the
  first-cut cost model.
- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md):
  the TTLs and write premiums behind w.
