# Cursor Harness: Probe Design and Effort Projection

Cursor is a probe-backed harness. Unlike Claude or OpenCode, where the model ID is
a simple string passed through, cursor bakes effort level and thinking mode into the
model slug itself — and does so inconsistently across model families. This page covers
the probe architecture, the raw-slug pattern that handles inconsistency, and the
catalog-driven effort projection that resolves the correct slug at launch time.

**Related pages:**
- [mars-routing.md](mars-routing.md) — routing architecture; probe-backed harness pattern; prefix boundary rules
- [mars-launch-bundle.md](mars-launch-bundle.md) — bundle `routing.harness_model` contract
- [mars-model-refresh.md](mars-model-refresh.md) — probe cache refresh modes for `build launch-bundle`
- [../concepts/model-resolution/aliases-and-routing.md](../concepts/model-resolution/aliases-and-routing.md) — meridian-side alias resolution
- [../lessons/harness-integration.md](../lessons/harness-integration.md) — stale mars binary / missing harness_model gotcha

---

## Build-Time Effort Resolution (Current)

**Mars** (launch-bundle build, `src/build/policy/runnable.rs`) resolves Cursor
`model` + `execution_policy.effort` into the exact probe slug via
`resolve_cursor_effort_slug` and writes it to **`routing.harness_model`**. On success,
`execution_policy.effort` is cleared (`effort_consumed`) so the harness receives a
single concrete slug.

Effort tier rules (same function as probe module):
- **`medium`**, **`none`**, **`auto`**, **`default`** → unsuffixed base slug when the
  probe lists it (Cursor’s default tier), not `model-medium`.
- Multiple matches at the same tier → prefer slugs containing **`thinking`** (shortest).

Meridian should pass **`routing.harness_model`** to `cursor agent --model` without
re-running the effort projector when the bundle was built with a current Mars binary.

**`routing.candidate_slugs`** is diagnostic-only on current Mars; do not treat it as
the launch authority for effort.

---

## Why Cursor Needs a Probe

Before cursor probe support, cursor was registered as `UniversalPassthrough` in
mars-agents. This meant:

- All cursor models reported `availability: unknown`
- `meridian spawn -m gpt-5.5` could not route to cursor by slug evidence
- Users needed explicit aliases with `harness = "cursor"` per model to avoid ambiguity

Cursor exposes `cursor agent --list-models`, which emits the full runtime catalog of
~100 slugs. Probe-backed routing uses this to confirm model availability and pass the
slug set to meridian for effort resolution.

---

## The Slug Naming Problem

Cursor's `--list-models` output illustrates the naming inconsistency:

```
gpt-5.5-high                       # base-effort
claude-opus-4-7-thinking-high      # base-thinking-effort
claude-4.6-opus-high-thinking      # base-effort-thinking  (different family, reversed order)
gpt-5.5-extra-high                 # two-word effort suffix
claude-opus-4-7-xhigh              # single-word effort synonym
gpt-5.1-codex-max-medium           # "codex-max" is the model family, not effort
```

There is no consistent rule for where effort appears relative to thinking mode, or
which spelling is used for the same concept. This makes slug decomposition fragile:
any parser with model-family-specific rules breaks when cursor adds new models.

**The raw-slug pattern:** Mars stores the complete `Vec<String>` of raw slug IDs and
matches by prefix only. No structural decomposition. `gpt-5.5` matches `gpt-5.5-high`,
`gpt-5.5-low`, `gpt-5.5-extra-high`, etc. Exact match takes priority.

**Rejected alternative:** `CursorSlugEntry { raw_slug, base_model, effort }` —
parsing into structured entries would require model-family-specific rules that break
on new models. The raw-string approach is robust to naming evolution.

---

## Probe Implementation

**Location:** `src/models/probes/cursor.rs` (mars-agents)

```rust
pub struct CursorProbeResult {
    pub slugs: Vec<String>,          // raw slug strings, fast variants excluded
    pub model_probe_success: bool,
    pub error: Option<String>,
}
```

The probe runs `cursor agent --list-models`, strips ANSI codes, and parses the output
format:

```
Available models

slug-id - Display Name
...

Tip: use --model <id> to select
```

Parsing rules:
- Skip the "Available models" header and blank lines
- Split on ` - `, take the slug (first column), trim whitespace
- Skip any slug ending in `-fast` (deferred to mars-agents#256)
- Silently skip malformed lines

No structural decomposition of the slug is performed.

---

## Prefix Matching in Routing

**Location:** `src/routing/mod.rs` (mars-agents)

When routing a model ID against cursor's catalog:

1. **Exact match first** — if the model ID appears verbatim in the catalog (after
   case and `.`→`-` normalization), return `MatchEvidence::Confirmed` with the exact
   slug.
2. **Prefix match** — find all catalog slugs that start with the normalized model ID.
   Return `MatchEvidence::Confirmed` with `chosen_slug = model_id` (the prefix, not
   a specific variant). All matching slugs populate `candidate_slugs`.
3. **No match** — `chosen_slug = None`, `skip_reason = "no_model_match"`.
4. **No probe result** — return `MatchEvidence::Passthrough` (offline or first-run).

`chosen_slug` is always the base model ID (the prefix), never a specific effort
variant. Effort selection is a projection concern, not a routing concern.

Mars uses `candidate_slugs` internally for routing confidence but does not thread it to meridian. Meridian receives the already-resolved `harness_model` and passes it through.

---

## Probe Cache

**Location:** `src/models/probes/cursor_cache.rs` (mars-agents)

Mirrors the opencode and pi probe cache exactly:
- Cache file: `{cache_root}/availability/cursor-probe.json`
- Lock file: `{cache_root}/availability/.cursor-probe.lock`
- TTL: `MARS_PROBE_CACHE_TTL_SECS` (default 60s)
- Schema version: `1`
- Outcomes: `Hit(result) | Stale(result) | Miss(result) | Unavailable`
- Background refresh via `mars models __refresh-probe --target cursor`
- Cross-platform locking: `fcntl.flock` on Unix, `LockFileEx` on Windows

Three harnesses now share this probe cache pattern (opencode, pi, cursor). The
structural duplication is tracked in mars-agents#67 but deferred.

---

## Availability Classification

Mars classifies cursor model availability using `classify_cursor`:

| Scenario | Availability | Source |
|---|---|---|
| cursor not installed | `Unavailable` | `NoHarness` |
| cursor installed, no probe result | `Unknown` | `CursorProbeUnknown` |
| cursor installed, probe failed | `Unknown` | `CursorProbeUnknown` |
| cursor installed, probe ok, exact match | `Runnable` | `CursorProbe` |
| cursor installed, probe ok, prefix match | `Runnable` | `CursorProbe` |
| cursor installed, probe ok, no match | not routed | — |

`HarnessClass` changed from `UniversalPassthrough` to `ProbeBacked` in the registry
(`src/harness/registry.rs`). This is a semantic/documentation change — no runtime
code branches on this value for behavior.


---

## Deferred: Fast Variants

Slugs ending in `-fast` are filtered at parse time in `cursor.rs`
(`slug.ends_with("-fast")`). They never appear in availability, routing, or
`mars models list`. Tracked in mars-agents#256.
