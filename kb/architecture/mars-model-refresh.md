# Mars Model Catalog and Probe Refresh

How Mars keeps the **models.dev alias catalog** and **harness probe caches** fresh
before routing, `mars models list|resolve`, `mars sync`, and `mars build launch-bundle`.
Meridian operators invoke these flags on the **`mars`** CLI (or `meridian mars …` when
wired); Meridian does not yet expose equivalent spawn-time flags — see
[open-questions/mars-feature-gaps.md](../open-questions/mars-feature-gaps.md).

**Related pages:**
- [mars-routing.md](mars-routing.md) — routing evaluator, default **harness_order**, native catalog matching, deferred passthrough
- [mars-launch-bundle.md](mars-launch-bundle.md) — bundle `routing` fields; Cursor **harness_model** at build time
- [cursor-harness.md](cursor-harness.md) — Cursor probe and effort slug rules (shared with build-time resolution)
- [../concepts/package-management/sync-model.md](../concepts/package-management/sync-model.md) — `mars sync` pipeline and refresh flags

---

## Two Independent Axes

Refresh is not one switch. Mars separates:

| Axis | Type | What it controls |
|------|------|------------------|
| **Catalog** | `RefreshMode` inside `ensure_fresh` | `models-cache.json` (models.dev aliases) |
| **Probes** | `ProbeRefreshMode` | Pi / OpenCode / Cursor probe subprocesses and `__refresh-probe` background spawns |

CLI flags combine both into **`ModelsRefreshControl`** (`src/models/mod.rs`):

```rust
pub struct ModelsRefreshControl {
    pub catalog_mode: RefreshMode,      // Auto | Force | Offline
    pub probe_refresh: ProbeRefreshMode, // Background | Synchronous | Skip
}
```

`resolve_models_refresh_control(refresh_models, no_refresh_models)` is the single
parser for `--refresh-models` / `--no-refresh-models` on all supported subcommands.

---

## CLI Surfaces

| Command | `--refresh-models` | `--no-refresh-models` | Default (`ModelsRefreshControl::auto()`) |
|---------|-------------------|----------------------|------------------------------------------|
| `mars models list` | Yes | Yes | Catalog **Auto**, probes **Background** |
| `mars models resolve` | Yes | Yes | Same |
| `mars sync` | Yes | Yes | Same |
| `mars build launch-bundle` | Yes | Yes | Same |

Mutually exclusive: passing both flags is a config error.

**Out of scope (KB):** a global `mars --refresh-models` root flag and Meridian-side
`meridian spawn` / `meridian mars` passthrough for these flags — tracked as an open gap.

---

## Flag → Control Mapping

| Flags | `catalog_mode` | `probe_refresh` |
|-------|----------------|-----------------|
| (none) | `Auto` | `Background` |
| `--refresh-models` | `Force` | `Synchronous` |
| `--no-refresh-models` | `Offline` | `Skip` |

**`--refresh-models`:** force HTTP catalog fetch when needed; run stale/miss probe
refreshes **in-process** (no detached `mars models __refresh-probe` on stale cache).

**`--no-refresh-models`:** disk-only catalog (error if no usable cache); probes do not
run — but **stale probe cache files are still read** when present (`Skip` + installed harness).

---

## Catalog: `ensure_fresh`

**Location:** `src/models/mod.rs` — `ensure_fresh(mars_dir, ttl_hours, mode)`.

**Cache file:** `{project}/.mars/models-cache.json` (TTL from `settings.models_cache_ttl_hours`, default 24h).

### `RefreshMode` behavior

| Mode | Fresh cache | Stale / empty cache | Notes |
|------|-------------|---------------------|-------|
| **Auto** | Return cache (`AlreadyFresh`) | Fetch models.dev; on failure may return **stale** with `StaleFallback` | `MARS_OFFLINE=1` coerces Auto → **Offline** inside `ensure_fresh` |
| **Force** | Refetch | Refetch | Ignores `MARS_OFFLINE` for fetch decision |
| **Offline** | Return usable cache | Error `ModelCacheUnavailable` | Used for `--no-refresh-models`; error text mentions the flag when mode was Offline from that flag |

Launch-bundle policy resolution (`build/policy/mod.rs`) calls `ensure_fresh` with
`input.models_refresh.catalog_mode` instead of read-only cache load — default **Auto**
means HTTP only when TTL says stale.

On catalog failure during bundle build, Mars warns and falls back to best-effort
`load_models_cache`; routing still proceeds with whatever slugs exist.

---

## Probes: `ProbeRefreshMode`

**Location:** `src/models/probes/probe_refresh.rs` — shared by opencode, pi, and cursor caches.

| Mode | Stale usable cache | Miss / no cache | Background `__refresh-probe` |
|------|-------------------|-----------------|------------------------------|
| **Background** (default) | Return stale; spawn background refresh | Sync probe in-process | Yes on stale (unless offline) |
| **Synchronous** (`--refresh-models`) | Sync probe in-process | Sync probe in-process | Never |
| **Skip** (`--no-refresh-models`) | Return stale | **Unavailable** (no subprocess) | Never |

### `MARS_OFFLINE` vs `Skip`

| Mechanism | Catalog | Probes |
|-----------|---------|--------|
| **`MARS_OFFLINE=1`** | Auto → Offline (no HTTP) | `should_probe_*` false — no subprocess |
| **`--no-refresh-models`** | Explicit Offline mode | `ProbeRefreshMode::Skip` |

Both suppress network and probe subprocesses, but through different code paths.
`Skip` still serves **stale** probe JSON from disk; offline blocks cold probes entirely.

Probe TTL: `MARS_PROBE_CACHE_TTL_SECS` (default 60s). Cache paths under
`MARS_CACHE_DIR` or `~/.mars/cache/availability/{pi,opencode,cursor}-probe.json`.

---

## Consumers of Fresh Data

The same **`ModelsRefreshControl`** flows into:

1. **`mars models list|resolve`** — `cli/models.rs` + shared routing evidence assembly
2. **`mars sync`** — alias refresh at end of sync (`sync/mod.rs`)
3. **`mars build launch-bundle`** — `PolicyInput.models_refresh` before harness routing

Routing evidence (catalog slugs + probe results) is aligned across these entry points
so `mars models resolve` and launch-bundle see the same native/prefix match behavior.
See [mars-routing.md](mars-routing.md#routing-parity-pr-72).

---

## Meridian Impact

Meridian does not set Mars refresh flags today. Operators who need deterministic or
air-gapped behavior should run `mars build launch-bundle` (or `mars sync`) with
`--no-refresh-models` / `MARS_OFFLINE=1` **before** spawn, or upgrade the Mars binary
Meridian invokes.

Stale Mars without probe support still produces bundles without `candidate_slugs`;
current Mars (≥0.6.4) bakes Cursor effort into **`routing.harness_model`** at build
time — Meridian should prefer that field. Legacy effort projection in meridian-cli is
documented in [cursor-harness.md](cursor-harness.md#meridian-effort-projection-legacy).
