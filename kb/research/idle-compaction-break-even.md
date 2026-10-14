# Idle Compaction Break-Even

**Status: analysis, not a decision.** Worked out 2026-10-14 in the
tech-lead primary `spawn:p7743`, alongside PR #578. Nothing here is
implemented and the user has not chosen a direction. It is recorded so it is
not derived again. [Idle Notifications](../decisions/idle-notifications.md)
links it as an open question under the stretch and guard decision.

Question: when is an idle compaction cheaper than letting the cache go cold?
Today the only size check is `idle.min_compact_tokens` (default 40,000), guard
13 in the [ordered guards](../decisions/idle-notifications.md#d-idle-stretch-at-most-once--each-stage-fires-at-most-once-per-stretch).

## What the current floor actually does

- It compares **total** context size, not growth.
- Only Claude and Pi report context size. Codex and OpenCode send none, and an
  unknown size fails open, so the floor never applies there (OpenCode 2.x is
  already tracked in `src/meridian/lib/harness/.context/FUTURE`).
- A Meridian Claude primary starts near the floor before any conversation.
  Measured on the tech-lead profile: the first request of `spawn:p7743` was
  39,386 input tokens with a 525-character opener; two other primaries started
  at 39.4k and 41.8k. Most other sessions in this project start at 20–25k.
- So for primaries the 40k floor blocks almost nothing, and a freshly
  compacted primary (≈39k prefix + summary) is already back over it.

## Break-even

All costs in units of the base input-token price.

| Symbol | Meaning |
|---|---|
| C | context size at the compact deadline |
| P | fixed prefix (system prompt, tools, profile) |
| H | history, C − P |
| S | summary tokens produced by compaction |
| c | cached-read multiplier, ~0.1 |
| w | cold re-read on return: Anthropic cache write 1.25× (5 min) or 2× (1 h); OpenAI 1× (no write premium) |
| o | output multiplier: ~5× Claude, ~8× GPT-5.x |
| r | probability the user returns to the session |

Not compacting costs `r·w·C` (cold re-read on return). Compacting costs
`c·C + o·S` now plus `r·w·(P + S)` on return. Compaction pays iff

```
H > (o·S + r·w·S + c·P) / (r·w − c)
```

With P = 39k and S = 5k:

| Provider / TTL | r = 1 | r = 0.5 |
|---|---:|---:|
| Claude, 1 h cache | ≈ 20k | ≈ 38k |
| Claude, 5 min cache | ≈ 31k | ≈ 61k |
| OpenAI | ≈ 54k | ≈ 116k |

The threshold is on **H**, growth beyond the prefix. The current floor on total
C ≥ 40k equals H ≥ ~1k for a Claude primary, an order of magnitude below
break-even.

## Caveats

- **S dominates and is a guess.** It is measurable from Claude and Pi, which
  report context before and after a compaction.
- **The formula assumes the compaction request hits the warm cache.** If a
  harness compacts with a different system prompt, the prefix misses and c
  rises from ~0.1 to an uncached read (≥1×). On OpenAI, where w = 1, the
  denominator `r·w − c` is then ≤ 0: compaction never pays on cost.
- **Subscription plans bill in usage limits, not money.** How cache writes,
  cached reads and output weigh against those limits is unknown.
- The formula prices tokens only. Compaction loses detail, and a cold return
  is slower; neither is counted.

## Proposal on the table (undecided)

1. Floor on **growth since session start or since the last compaction**, not
   total size, so a primary's ~39k prefix doesn't count.
2. Set the floor **per provider** from the formula instead of one 40k default.
3. Make Codex and OpenCode report context size so any floor applies there.
4. **Measure S live first**; every threshold above moves with it.

## Related

- [Prompt-Cache Retention by Harness](prompt-cache-retention.md): the TTLs and
  write premiums that set w.
- [Idle Notifications](../decisions/idle-notifications.md): the schedule and
  guard list this would change.
- [Idle Adapter Architecture](../architecture/idle-adapters.md#per-harness-coverage):
  which adapters report context size.
