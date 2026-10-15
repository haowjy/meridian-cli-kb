# Decision: Idle Compaction Only When It Pays

## D-idle-compaction-growth-floor — Compact on growth past a per-harness break-even

**Status:** decided by the user (2026-10-14, "2 yes do it"; review cuts
approved after `spawn:p7956`). Merged to `main` in PR #589 (`df0f9f954`,
2026-10-15) as a patch release (v0.10.6). It replaces `idle.min_compact_tokens`
and the total-context guard that every release through v0.10.5 applied.

Provenance: `work:idle-compaction-floor` (design `design/compaction-floor.md`,
approved cuts `DIVERGENCE/review-cuts.md`, measurements in `evidence/`) and
`work:harness-compaction-cache` (OpenCode 1.x/2.x cache split, Pi research).
Measurements `spawn:p7919`, cache probe `spawn:p7933`, design `spawn:p7934`
reviewed by `spawn:p7942`, implementation `spawn:p7943`, strict review
`spawn:p7956`, fixes `spawn:p7958`, OpenCode 2.x pricing `spawn:p7979` and
seam cut `spawn:p7985`. First-cut break-even analysis: tech-lead primary
`spawn:p7743`.

### Problem

Idle compaction exists to make a cold return cheap
([Idle Notifications](idle-notifications.md)). The old guard skipped only when
a *known* total context was under a fixed 40,000 tokens, and an unknown context
compacted. It failed in three ways:

- A Meridian Claude primary starts at about 39k tokens before any conversation
  (the tech-lead profile's first request measured 39,386), so the floor blocked
  almost nothing. A primary that had just been compacted (about 44–52k)
  qualified again in every stretch and re-summarized its own summary.
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

C is context now, B the baseline, R the compaction residue (summary plus
whatever the harness keeps), w the cold re-read multiplier on return, c and u
the compaction request's read multiplier and flat uncached input, o and S the
output multiplier and summary size, and r the probability the user returns.
Growth is an integer and the inequality is strict, so the code uses
`need = floor(x) + 1` and checks `growth >= need`. `lib/idle/worth.py` owns
the formula, and the "not worth it" guard calls it.

**Growth, not total (rejected: a better total floor).** The fixed prefix is
paid either way, so only history past the baseline can be saved. Any total
floor either blocks fresh primaries or lets a post-compaction primary
re-qualify every stretch; no single number does both jobs.

**Deliberately left out:** after a return, later turns re-read `C_after`
instead of C (favours compaction), and compaction loses detail (counts against
it). Neither is priced; they roughly cancel, and leaving both out keeps the
rule to token prices. Subscription plans bill usage limits rather than tokens,
with unknown weights; the override covers them.

**Defaults at r = 0.5:**

| Route | Costs in code (w, c, u, o, S, R) | need (B from start or a shrink / B after a Meridian compaction) | Default |
|---|---|---:|---|
| Claude, 1 h TTL | 2, 0.1, 20,000, 5, 4,000, 13,000 | ≈63k / ≈50k | compacts when grown enough |
| Claude, any other TTL | 1.25, rest as above | ≈99k / ≈86k | a 5 min TTL has no compact stage anyway |
| Codex (assumes the OpenAI/ChatGPT route) | 1, 0.114, 0, 8, 4,500, 3,500 | ≈103k / ≈100k | compacts only after large growth (o·S dominates) |
| OpenCode 2.x `openai` | 1, 0.104, 1,000, 8, 1,600, 16,500 | ≈60k / ≈44k | compacts when grown enough |
| Pi `anthropic` long retention, `openai`, `openai-codex` | none: the bundle port is absent | never | `skipped:not-worth-it`; push and warn unchanged |
| OpenCode 1.x; OpenCode 2.x and Pi on other providers | — | — | no detected TTL, so no compact stage: push only |

Where the constants come from (design §2): Claude c and u fit two Haiku probe
runs that matched `0.1·C + 20k` exactly, and a realistic primary (live probe
P8) had 19.3k uncached input, under the 20k assumed. Codex c is the median of
160 local `token_usage_record`s before a `compacted` record. OpenCode 2.x c is
one OpenAI compaction that read 187,520 cached tokens and 730 uncached; its
compaction reuses the session system prompt, tools and older native messages
and appends the summary instruction, so it is a prefix of a normal request.

**Which routes never pay, and why.** A route never pays when its compaction
request reads cold (c = 1) and the return costs at most `w = 2`: at r = 0.5,
`r·w ≤ c`. Two harnesses compact cold by construction:

- **Pi** summarizes with its own system prompt, flattens the conversation into
  one user message and sends `cacheRetention: "none"`, so no cache read is
  possible. Measured cold on `openai-codex`.
- **OpenCode 1.x** compacts with an empty system prompt and no tools. All 109
  locally measured OpenCode compactions with a 0% cache read were 1.x rows.

An earlier version of this record said OpenCode never pays on every version.
That came from the same 109 rows, mislabelled as 2.x; 2.x stores its
compactions separately, and its 11 rows read 93–99.8% from cache
(`work:harness-compaction-cache`, `evidence/opencode-cache-by-provider.md`).
OpenCode 1.x needs no cost entry: it reports no provider (so no TTL and no
compact stage) and no context.

**Never-paying routes keep the stage.** The compact stage stays on the timeline
and the guard skips it. Dropping it from `timeline.schedule` would put a second
copy of the rule outside the ordered guards and lose the persisted evaluation
that explains the skip.

**Unknown now fails closed.** An unknown context gives `skipped:context-unknown`,
and a missing baseline gives `skipped:baseline-unknown`. Like every other
non-transient compact skip, these last for the rest of the stretch. This
replaces the old rule that unknown context fails open
([D-idle-stretch-at-most-once](idle-notifications.md#d-idle-stretch-at-most-once--each-stage-fires-at-most-once-per-stretch)).

### Cuts the user approved after review

The design's state and cost-port shape were narrowed before shipping (review
`spawn:p7956`, applied in `spawn:p7958`):

- **r is fixed at 0.5**: `RETURN_PROBABILITY` in `worth.py`. It is not config,
  and no caller passes it. `CompactEval` still records `r`, so old evaluations
  stay readable after it changes.
- **Costs live on the harness bundles, and only where they change a verdict.**
  Claude, Codex and OpenCode register the `compact_costs` port. Pi costs gave
  the same verdict as no costs at r = 0.5, so they were deleted (YAGNI); the
  measured constants stay in design §2. At r = 1, Pi `anthropic` long retention
  would pay after about 70k of growth.
- **No persisted provider.** Codex never receives a provider (its notify has
  none). Its costs assume OpenAI; a Codex session on another `model_provider`
  gets OpenAI costs, so use the override there. OpenCode's costs likewise
  apply only where a compact stage exists, which today is the `openai`
  provider. TTL detection still gets the provider from the arm arguments.
- **Baseline is a frozen `ContextBaseline` under `IdleState.context`**
  (`context_tokens`, `baseline_tokens`, `baseline_source`). Two pure methods,
  `observe(x)` and `after_compaction()`, replace a flat set of fields that
  every arm branch had to copy by hand.
- **`compact_eval` is scoped to one stretch.** It survives re-anchors and
  resets when a stretch opens, because it explains that stretch's
  `done.compact`.

**Rejected after the OpenCode correction: a sensor-level cost seam.** The first
OpenCode 2.x fix (`spawn:p7979`) added a `compact_costs` protocol on the sensor
so that V1 would stay "never". It was redundant: V1 has no TTL, so it never
reaches the guard. It was deleted (`spawn:p7985`) and OpenCode uses the same
bundle port as Claude and Codex.

### Pi stays "never" until in-context compaction ships

Pi's cold compaction is a request-shape problem in Pi, not a provider problem.
Upstream PR earendil-works/pi#10577 adds an opt-in `compaction.inContext`
mode that repeats the normal request and appends the summary instruction, gated
to a verified model list (Claude, `openai-codex` gpt-5.6/gpt-6 models,
selected `opencode-go` models) and a warm-cache check.

Decision (user, 2026-10-15): keep Pi at "never" and do not build a
Meridian-extension version (Pi's `session_before_compact` hook returning a
Meridian-built summary). It is feasible but would duplicate #10577. When
#10577 ships, enable `compaction.inContext` for Meridian Pi sessions and give
Pi costs only for verified models: haowjy/meridian-cli#593. The user asked not
to comment on the upstream PR on their behalf.

Pi 0.86.0's idle cache warming does not conflict today
(`research/pi-cache-warming.md`): Pi's catalog declares no cache lifetime for
OpenAI routes, and on Anthropic long retention Pi's 30-minute idle horizon ends
before its first refresh (54 min). See
[Prompt-Cache Retention](../research/prompt-cache-retention.md#pi-cache-warming).

### Baseline lifecycle

| Event | Effect on `IdleState.context` |
|---|---|
| arm with context X > 0 (new stretch, re-anchor, `already-compacted`, duplicate turn) | B pending → B = X, source kept. X < last context (shrink: native compaction, user `/compact`, rewind) → B = X, source `observed`. Then context = X. |
| arm absorbed by `compact-window` or `expected-compaction-turn` | no change |
| **first arm after a closed stretch whose `done.compact` is `claimed` or `ok`** | **no change** |
| arm with no context, or 0 | no change |
| `done compact ok` | B pending, source `compacted` (the next real observation sets B with Δ = 0) |
| `done failed` / `vetoed`, return, `close_on_launch`, new stretch | no change: the baseline outlives stretches |

**Why the closed-stretch skip exists (review S1).** A return or reopen during
a Claude compaction clears the compaction window and the expected-turn flag.
The compaction's own turn end then arrives as an ordinary arm on a closed
stretch. If Claude reports the compacted `postTokens` (about 1.2k) at that
moment, B collapses and the next real turn looks like 50k of growth, so
Meridian compacts again. Spuriously small observations are the one way the
rule over-compacts. Codex (`input_tokens == 0`, `compacted`), OpenCode (the
compaction message) and Pi (`null`) already report nothing there. Skipping
that arm costs one observation, and the next turn sets B. Service tests cover
both orders. **Not confirmed:** whether native Claude actually emits
`turn.complete` for a compaction after a `prompt.submit`; the probe drove the
CLI with injected arms.

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
| Codex | rollout tail, `token_count.info.last_token_usage.total_tokens` (`codex_rollout.latest_context_tokens`). Read at notify and in sensor `facts()`. `None` at a `compacted` record or `input_tokens == 0` |
| OpenCode 2.x | newest assistant message in `/api/session/{id}/message`: `tokens.input + cache.read + cache.write`. Read at turn end and again at fire. A compaction message (`mode == "compaction"` or `summary`) stops the scan **before** the zero-usage check, so a compaction still in progress returns `None` |
| OpenCode 1.x | none |

**Both reads stay (rejected: fire reads persisted state only).** An arm without
context deliberately leaves `context_tokens` unchanged, so a later shrink is
still visible. After a turn in which the harness compacted natively, state still
holds the old, larger size. The fire-time re-read is what turns that into
`context-unknown`. A fire that trusted state would compact a session that was
just compacted. The re-read also corrects Codex's late `token_count` flush.

### Config and diagnostics

- `idle.compact_min_growth_tokens`, overridable per harness under
  `[harness.<h>.idle]`, replaces `idle.min_compact_tokens`. The env vars are
  `MERIDIAN_IDLE_COMPACT_MIN_GROWTH_TOKENS` and
  `MERIDIAN_HARNESS_IDLE_COMPACT_MIN_GROWTH_TOKENS_<H>`. Unset means break-even.
  An integer replaces `need` (`basis: configured`), ignores the costs, and so
  re-enables Pi. It still needs a known context and baseline. `0` compacts on
  any known growth.
- Rejected: an `r` setting, whose effect the user can't see (status shows
  `need`, which is what the override replaces); and `compact = "auto"`, which
  can't tune a threshold and would change an existing key's type.
- `meridian idle status --json` shows `context` and `compact_eval` (context,
  baseline, growth, need, `basis` of `break-even`/`configured`/`never`, `r`,
  costs, verdict). The table shows `skipped:not-worth-it (19k/50k)`. The `idle
  fire` wire adds `detail`, for example `growth 19k < need 50k (break-even)`.
  A guard skip sends no notification.

### Released as a patch, despite the key rename

`docs/releasing.md` reserves `release:minor` for state an older build cannot
read, and the PR body first suggested minor: the config key is renamed with no
alias, and v0.10.0 cannot read the new idle rows (its `idle status` returned an
empty list against copies). The user chose patch anyway ("no, i would prefer
not to minor"; every PR in this stack stayed `release:patch`). No reason was
given beyond preference; the project has no external users and keeps no
backwards compatibility. The upgrade note in `docs/upgrading.md` and the
CHANGELOG name the rename; a leftover old key no longer has any effect.
Rollback means restoring the old key and not running both
versions against one idle directory.

### Verification

- Live probes P1–P8 ran against the pre-cut implementation (`spawn:p7943`,
  `evidence/floor-live.md`). P1 matched the persisted context to each harness's
  native usage. P5 exercised the OpenCode parser on a free provider because
  the isolated `openai` route was unavailable.
- After the cuts: focused tests, ruff and pyright clean; installed-wheel CLI
  checks covered both completion orders and the cost ports. OpenCode 2.x
  pricing was checked against the threshold arithmetic (60,410 need from an
  18k start, 44,041 after a Meridian compaction); its compaction under the
  break-even rule has not fired live.

### Revisit when

- **r moves.** It is a judgement: 0.5 against a measured 0.27 that is biased
  low, because state keeps one stretch per file. Claude's need roughly doubles
  across the plausible range. Measuring r conditioned on reaching the deadline
  needs a per-stretch outcome record, which is deferred. If r rises, restore
  the Pi costs from design §2.
- **Pi #10577 ships** (haowjy/meridian-cli#593).
- **Claude's uncached input passes 20k** on real primaries: each +15k of u adds
  about 17k of need.
- **Native Claude ordering** of a compaction turn end after a return is
  observed live.
- **Codex is routed to a non-OpenAI provider**, or OpenCode 2.x gains a known
  TTL on another provider, in practice.

## Related

- [Idle Notifications](idle-notifications.md): the schedule, guard order and
  stretch rules this changes.
- [Idle Adapter Architecture](../architecture/idle-adapters.md#compaction-worth): the
  current adapter mechanism.
- [Prompt-Cache Retention by Harness](../research/prompt-cache-retention.md):
  the TTLs and write premiums behind w, and which compactions read the cache.
