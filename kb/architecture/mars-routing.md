# Mars Routing Architecture

Internal routing system inside mars-agents. Determines which harness + model pair
handles a given agent invocation. Meridian consumes the routing output via the
launch bundle `routing` object; it never sees the internal Rust types.

**Related pages:**
- [mars-launch-bundle.md](mars-launch-bundle.md) — how routing results land in the bundle `routing` object
- [../concepts/model-resolution/aliases-and-routing.md](../concepts/model-resolution/aliases-and-routing.md) — meridian-side alias resolution and harness selection

---

## Module Structure (post-PR #58 refactor)

```
src/
  routing/
    mod.rs          # evaluate_candidates(), evaluate_fixed_harness() — public evaluator
    slug.rs         # shared slug-matching primitive (SlugParts, find_model_matches, find_exact_match)
    acceptance.rs   # accept_route() + accept_assessment() — policy acceptance layer
    report.rs       # RouteDecisionReport — serialization DTO
  models/
    availability.rs # thin consumer of routing::slug
    probes/
      pi_cache.rs   # SCHEMA_VERSION 2 (bumped to invalidate legacy entries)
  config/
    routing_settings.rs  # ResolvedRoutingSettings — CLI typed policy boundary
  build/
    bundle.rs            # Routing struct uses RouteDecisionReport fields
    policy/
      harness.rs         # uses acceptance layer + RouteDecisionReport
      runnable.rs        # uses RouteDecisionReport, no internal-type digging
  cli/
    models.rs            # ResolvedRoutingSettings consumer
```

## Dependency Direction

Stable → volatile:

```
routing::slug
  └── routing::mod (evaluator)
        └── routing::acceptance
              └── routing::report
                    └── build::policy::*, cli::models
```

`routing::slug` has no internal dependencies. `models::availability` depends on
`routing::slug` (not vice versa). All consumer modules depend on report/acceptance,
not on evaluator-internal types.

---

## Shared Slug Primitive (`routing::slug`)

**Why this module exists:** `routing/mod.rs` and `models/availability.rs` each had
duplicated slug-matching logic. Extracted into `routing::slug` so there is one
implementation.

**`SlugParts<'a>`** — borrows from the input string, no allocation:

```rust
struct SlugParts<'a> {
    provider: &'a str,
    model:    &'a str,
    variant:  Option<&'a str>,
}
```

Hot path: slug scanning in `select_probe_slug` and `find_matching_slug` parses
and discards. Borrowing avoids allocation there. Owned `DecomposedSlug` replaced.

**`SlugMatch`** — returned by `find_model_matches` / `find_exact_match` — produces
owned strings for callers that store results.

**Placement rationale:** `routing::slug` not `models::slug`. Primary consumer
is the evaluator. `models::availability` already depended on routing; putting slug
in models would invert that dependency direction.

---

## Evidence Split: `SelectionKind` + `MatchEvidence`

Before PR #58, `RouteConfidence` was a single enum conflating two orthogonal concerns:
- **How was the harness selected?** (user forced it, evaluator picked it)
- **What slug evidence exists?** (exact match, prefix match, none)

The `Forced` variant was the symptom: callers set `confidence = Forced` to signal
"user chose this harness," overwriting the actual slug evidence. Consumers then
had to dig into nested internals to recover the evidence they lost.

**Post-refactor: two orthogonal enums**

```rust
enum SelectionKind {
    ... // evaluator-selected vs caller-fixed selection mode
}

enum MatchEvidence {
    ... // slug/evidence quality categories
}
```

Fixed-harness callers set `SelectionKind::Fixed` and preserve the actual
`MatchEvidence` from the assessment instead of overwriting it. Consumers that
previously checked `confidence == Forced` now check `selection_kind == Fixed`.

---

## Acceptance Layer (`routing::acceptance`)

Five scattered open-coded acceptance checks replaced with two shared functions:

```rust
// Trace-level: 3 call sites
fn accept_route(trace: &RouteTrace, policy: &RoutingPolicy) -> Result<(), RejectionReason>

// Assessment-level: 2 call sites
fn accept_assessment(assessment: &RouteAssessment) -> Result<(), RejectionReason>
```

**Design:** Not one function with a trait. Two functions share `RejectionReason`.
Trivially simple at 5 call sites — a trait would be over-engineering.

**Evaluator-pure design:** The evaluator (`mod.rs`) never errors on routing failures.
It scores candidates and returns a trace. The acceptance layer is where callers
apply policy. Separation keeps policy out of the scoring algorithm and makes the
evaluator independently testable.

---

## Serialization DTO (`routing::report::RouteDecisionReport`)

**Mars-internal DTO.** This is the public surface for consumers _within_ mars-agents (outside the routing module). Meridian never sees this type — it consumes only the fields in the bundle's `routing` JSON object.

**String labels for diagnostic fields:**

```rust
struct RouteDecisionReport {
    harness:         String,   // not HarnessId enum
    model_id:        String,
    selection_kind:  String,   // Mars-internal diagnostic label
    match_evidence:  String,   // Mars-internal diagnostic label
    ...
}
```

Rationale: serialized diagnostics are decoupled from internal enum variants and can
evolve without changing Meridian's consumed routing contract.

**Mars-internal consumers only:** `build::policy::harness`, `build::policy::runnable`, and `cli::models` are all Mars modules. Meridian's bundle adapter (`bundle_adapter.py`) reads only `routing.model`, `routing.model_token`, `routing.harness`, and `routing.harness_model`. Diagnostic routing labels and route traces are Mars-owned instrumentation and are ignored by Meridian.

---

## Pi Cache Schema Version Bump

`models/probes/pi_cache.rs` — `SCHEMA_VERSION` bumped 1 → 2.

Legacy cache entries are missing `model_slugs`. Three options considered: bump
version, add `supports_model_slugs: bool`, check slug presence in
`is_usable_result()`. Version bump wins: `read_cache_tolerant_at()` already
rejects version mismatches — zero new code, and cache validation stays unaware
of routing concerns.

---

## CLI Typed Policy Boundary

`config/routing_settings.rs` defines `ResolvedRoutingSettings`. Before this
refactor it was defined but never called; `cli/models.rs` had a private
`RoutingSettings` duplicate instead.

`ResolvedRoutingSettings` is now wired into the CLI. Build pipeline stays on raw
slices — it has its own overlay merge logic, forcing typed resolution there would
duplicate validation or restructure the merge. Typed resolver at CLI boundary;
raw slices inside the build pipeline.

---

## Phase Order (PR #58)

Sequenced as four independent compilation units:

| Phase | Work | Rationale |
|-------|------|-----------|
| 1 | `routing::slug` + pi cache bump | Creates shared primitive; clears cross-module coupling |
| 2 | Wire `ResolvedRoutingSettings` to CLI | Clears dead code before bigger changes |
| 3 | `SelectionKind + MatchEvidence` split + `RouteDecisionReport` | Coupled — DTO needs new confidence split |
| 4 | `routing::acceptance` | Depends on types from Phase 3 |

Each phase compiles and passes tests independently.
