# Idle Compaction Break-Even

**Status: first-cut analysis, superseded by a decision.** Worked out
2026-10-14 in the tech-lead primary `spawn:p7743`, alongside PR #578. The user
then chose a growth-based, per-harness break-even:
[D-idle-compaction-growth-floor](../decisions/idle-compaction-growth-floor.md#d-idle-compaction-growth-floor--compact-on-growth-past-a-per-harness-break-even).
That record owns the rule, the measured constants and the status. It is
unreleased, on a local branch. This page keeps the first derivation and the
evidence about the old floor.

Question: when is an idle compaction cheaper than letting the cache go cold?
On `main` the only size check is `idle.min_compact_tokens` (default 40,000),
guard 13 in the [ordered guards](../decisions/idle-notifications.md#d-idle-stretch-at-most-once--each-stage-fires-at-most-once-per-stretch).

## What the old floor does (still current on `main`)

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

## What the decision changed from this analysis

Measuring before designing (`work:idle-compaction-floor`) refined the model
in three ways:

- **A flat uncached term `u`.** Claude's compaction request fits
  `0.1·C + 20k`, not a plain multiplier, so the rule prices `c·C + u`.
- **The baseline stands in for C_after.** After Meridian's own compaction, the
  baseline already includes the residue. Any other baseline (session start, a
  rewind, a native compaction) adds a measured residue R. This replaces the
  P + S of the first cut.
- **The warm-cache caveat came true on some routes.** Pi and OpenCode
  compaction requests read cold (c = 1), so with w ≤ 2 they never pay at
  r = 0.5. Codex measured c ≈ 0.114 and pays only after about 100k of growth.

All four of this page's original proposals were adopted: floor on growth
rather than total size, set the floor per provider from the formula, make
Codex and OpenCode report context size, and measure S first. Context reporting
became required, because an unknown context now skips compaction instead of
allowing it. The final thresholds at r = 0.5 are higher than the
table above: Claude 1 h needs about 63k of growth from start and about 50k
after a Meridian compaction.

## Related

- [Prompt-Cache Retention by Harness](prompt-cache-retention.md): the TTLs and
  write premiums that set w.
- [Idle Notifications](../decisions/idle-notifications.md): the schedule and
  guard list this would change.
- [Idle Adapter Architecture](../architecture/idle-adapters.md#per-harness-coverage):
  which adapters report context size.
