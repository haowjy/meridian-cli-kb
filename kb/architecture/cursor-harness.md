# Cursor Harness: Probe Design and Effort Projection

Cursor is a probe-backed harness. Unlike Claude or OpenCode, where the model ID is
a simple string passed through, cursor bakes effort level and thinking mode into the
model slug itself — and does so inconsistently across model families. This page covers
the probe architecture, the raw-slug pattern that handles inconsistency, and the
catalog-driven effort projection that resolves the correct slug at launch time.

**Related pages:**
- [mars-routing.md](mars-routing.md) — routing architecture; probe-backed harness pattern
- [mars-launch-bundle.md](mars-launch-bundle.md) — launch bundle structure; `candidate_slugs` threading
- [../concepts/model-resolution/aliases-and-routing.md](../concepts/model-resolution/aliases-and-routing.md) — meridian-side alias resolution
- [../lessons/harness-integration.md](../lessons/harness-integration.md) — stale mars binary / empty candidate_slugs gotcha

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

`candidate_slugs` in the `CandidateAssessment` carries all prefix-matching raw slugs
forward through the launch bundle for meridian's effort projector to search.

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

## Effort Projection

**Location:** `src/meridian/lib/harness/projections/project_cursor.py` (meridian-cli)

Mars threads all prefix-matching raw slugs as `candidate_slugs` through the launch
bundle to meridian's projection layer. The projector searches them to find the best
slug for the resolved `(model, effort)` pair.

### Algorithm: `_resolve_cursor_model(model, effort, candidate_slugs)`

1. **Exact match** — if `model` appears verbatim in `candidate_slugs` and no effort
   override is specified, return it as-is (user passed a full cursor slug).
2. **No effort** — return `model` unchanged. Cursor picks its default effort.
3. **Effort specified** — search `candidate_slugs` for slugs where
   `_prefix_matches(slug, model)` (`slug == model or slug.startswith(f"{model}-")`) and
   either `slug.endswith(f"-{effort}")` (trailing) or `f"-{effort}-" in slug` (infix, for
   thinking variants like `model-high-thinking`).
   - One match → use it.
   - Multiple matches → prefer slugs containing `"thinking"` (shortest thinking match).
     If no thinking variant, pick the shortest overall. The length tiebreaker is the
     real defense against overmatch: `endswith("-high")` matches both `gpt-5.5-high` and
     `gpt-5.5-extra-high`; `min(key=len)` selects `gpt-5.5-high`.
   - No matches → fall back to `f"{model}-{effort}"` (let cursor reject if invalid).

### Thinking preference and length tiebreaker

When multiple effort candidates exist:

1. **Thinking preferred** — if any candidate contains `"thinking"`, the shortest such
   slug is returned. This is cursor-harness policy baked into projection.
2. **Length tiebreaker** — if no thinking variants exist, `min(key=len)` selects the
   shortest match. This is the mechanism that correctly handles `endswith` overmatch:
   both `gpt-5.5-high` and `gpt-5.5-extra-high` satisfy `endswith("-high")` when
   `effort="high"`, but `gpt-5.5-high` is shorter and wins. Note: if the catalog only
   contains `gpt-5.5-extra-high` and `effort=high` is requested, that slug would be
   selected — the tiebreaker only helps when a precise match also exists.

### Behavior table

| `spec.model` | `spec.effort` | Result |
|---|---|---|
| `gpt-5.5` | `None` | `gpt-5.5` |
| `gpt-5.5` | `high` | `gpt-5.5-high` (from catalog) |
| `claude-opus-4-7` | `high` | `claude-opus-4-7-thinking-high` (thinking preferred) |
| `claude-4.6-opus` | `high` | `claude-4.6-opus-high-thinking` (thinking preferred) |
| `claude-opus-4-7-thinking-high` | `None` | `claude-opus-4-7-thinking-high` (exact match) |
| `gpt-5.5` | `ultra` | `gpt-5.5-ultra` (fallback, no catalog match) |
| `gpt-5.5` | `high` | `gpt-5.5-high` (fallback, empty candidate_slugs) |

### Field movement

`effort` moved from `_DELEGATED_FIELDS` to `_PROJECTED_FIELDS` in the cursor
projector. Cursor effort resolution happens in the projector, not by delegation to
the harness.

---

## candidate_slugs Threading (Two-Repo Flow)

```
mars routing evaluates model prefix against cursor catalog
    → populates candidate_slugs in CandidateAssessment
    → included in launch bundle routing object

meridian reads candidate_slugs from launch bundle
    → populates ResolvedLaunchSpec.candidate_slugs
    → passed to project_cursor_spec_to_cli_args()
    → _resolve_cursor_model() searches them for best effort match
```

The launch bundle routing object gained `candidate_slugs` as a new field for cursor.
When `candidate_slugs` is empty (offline, old mars binary, no probe result), the
effort projector falls back to blind suffix construction (`f"{model}-{effort}"`).

**Critical dependency:** If the mars binary predates cursor probe support, the
routing result will not include `candidate_slugs`, and effort projection silently
falls back to blind construction. See the lesson in
[../lessons/harness-integration.md](../lessons/harness-integration.md).

---

## Deferred: Fast Variants

Slugs ending in `-fast` are filtered at parse time in `cursor.rs`
(`slug.ends_with("-fast")`). They never appear in availability, routing, or
`mars models list`. Tracked in mars-agents#256.
