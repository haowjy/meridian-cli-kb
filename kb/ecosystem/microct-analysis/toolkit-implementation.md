# MicroCT Analysis — Generic Toolkit Implementation

Status: Shipped and validated (2026-05-05)  
Implementation: 6 phases, 34 EARS statements, 7 refactors, 18 spawns  
Validation: OA6-1RK — all 6 geometric indices passed ±10% gate  
Package architecture: [overview.md](./overview.md)  
Measurement design: [measurement-design.md](./measurement-design.md)

---

## Design Philosophy: Domain-Agnostic Tools

The `microct-analysis` package is a **generic volumetric image analysis toolkit** — not a mouse-knee OA tool. The Python tools layer has no knowledge of anatomy, species, pathology, or measurement protocol. Domain knowledge enters exclusively through workflow definition files at runtime.

This means the same tools (`segmentation_tools.py`, `orientation_tools.py`, `preprocess_tools.py`, etc.) can support different anatomies and protocols. A new workflow file is all that's required to apply the toolkit to a new anatomy or study design.

**Rejected alternative: hardcode mouse-knee anatomy into tools.** Embedding landmark names, expected component counts, and threshold values into tool implementations would couple the toolkit to a single protocol and make every new study a code change. The workflow file pattern separates what to measure from how to measure.

---

## 3DMedAgent Pattern: Progressive Narrowing

The slice-examination loop is based on the **3DMedAgent paper pattern** (OAMI → CFLT → T1S-Loop):

- **OAMI** — Overview scan: render overview slices across all three planes to understand gross anatomy
- **CFLT** — Candidate Feature Location Targeting: identify candidate regions from the overview
- **T1S-Loop** — Targeted 1-Slice Loop: iteratively refine at specific locations until confidence is achieved or futility is declared

This pattern drives the `slice-examination-loop` skill. The agent progressively narrows from full-volume overview to targeted slice examination before committing to decisions like seed placement. Rendering first — acting second — is not a preference but a requirement: seeds placed without visual evidence are unreliable.

---

## Slice-Examination-Loop: Skill, Not Agent

The slice-examination loop is a **skill** loaded into the segmenter agent, not a separate agent spawn.

**Why skill, not agent:**

- The loop operates on live kernel state — it needs access to the existing numpy arrays (`volume`, `bone_mask`) in the analyst's kernel session. A separate agent would need to reload state, breaking the kernel continuity guarantee.
- The loop is short-lived and iterative. Spawning an agent per iteration is expensive and loses the accumulated visual context between render cycles.
- Seeds placed inside the loop must be directly available for the subsequent `watershed_from_markers` call in the same execution context.

**Rejected alternative: separate `microct-slice-examiner` agent.** Would require state serialization in and out of the kernel, cross-agent coordination for seed handoff, and round-trip spawn overhead per iteration. Net effect: more complexity, less context.

**Skill shape:** The `slice-examination-loop` skill implements the render→examine→reason→act→render cycle in one agent context. It exports a `seed_evidence.json` contract for durable record of visual decisions.

### No Max Iterations

The loop has **no hardcoded iteration limit**. The agent uses judgment to determine convergence vs futility:

- **Convergence:** watershed result shows clean bone separation matching expected structure count
- **Futility:** repeated attempts across multiple planes yield the same failure mode with no anatomical resolution path

A max-iterations cap was explicitly rejected because it creates a false sense of completeness — an agent that exhausts its limit without resolution should report futility, not silently accept a bad segmentation. Futility is a legitimate and valuable output.

---

## Watershed Failure Root Cause and Fix

### Root Cause

Prior watershed implementations failed because `seed_from_region()` flood-filled seeds to entire connected components. When bones are partially bridged (touching at one point), flood-fill produces one giant seed spanning both bones. Watershed then has no information to separate them — the "seed" for each bone covers most of both bones.

### Fix: Sphere Markers

New function: `create_sphere_markers(volume_shape, points_zyx, labels, radius_voxels)`.

Sphere markers are small, spatially bounded seeds that don't expand. They rely on explicit anatomical point placement (from the slice-examination loop) rather than automatic region detection. Watershed then has clean separation between seeds and can propagate correctly through bridged regions.

```
Old path: seed_from_region() → flood-fill → giant overlapping seeds → watershed failure
New path: render → examine → place points → create_sphere_markers → watershed_from_markers → clean separation
```

**Why this fixes the failure:** Watershed is fundamentally a seeded algorithm. The quality of the output is bounded by the quality of the seeds. Small sphere markers placed at anatomy-informed locations provide tight, non-overlapping seeds. The flood-fill approach undermined the algorithm's basic requirement.

### `watershed_from_markers` vs `run_segmentation`

The package previously had `run_segmentation()` in `stages/segmentation.py` that bundled median_filter + threshold + labeling + watershed into one call. This was replaced by composable primitives:

- `threshold_mask()` — threshold only
- `component_summary()` — inspect components before acting
- `select_component()` — bounded selection from components
- `create_sphere_markers()` — sphere seeds from points
- `watershed_from_markers()` — watershed from markers

**Why composable:** `run_segmentation()` has peak memory across all steps (all arrays live simultaneously). The composable path lets the agent inspect intermediate state, apply the compute gate before each expensive step, and retry individual steps without re-running the full pipeline.

---

## CoordinateFrame Contract

Orientation tools produce a `CoordinateFrame` object that tracks coordinate frame metadata through the pipeline:

```python
@dataclass
class CoordinateFrame:
    name: str                           # "original" | "oriented"
    rotation_matrix: np.ndarray | None  # 3x3 rotation applied
    center_physical: np.ndarray | None  # origin of rotation in physical coords
```

**Why explicit frame tracking:** When the tibia is PCA-oriented, slice indices in the oriented volume do not correspond to slice indices in the original volume. Seed points placed in the oriented frame must be transformed back before writing `seed_evidence.json`. Without an explicit frame contract, frame confusion silently produces wrong coordinates.

The contract is enforced at the artifact boundary: `seed_evidence.json` must record `coordinate_frame` and `orientation_transform`. Downstream consumers know whether they need to invert the transform before using the points.

---

## Multi-Plane Rendering

The tools layer now supports axial, sagittal, and coronal rendering. `render_slice()` and `inspect_plane()` accept `plane="axial" | "sagittal" | "coronal"` (extended from axial-only).

**Why multi-plane:** Bridged-bone cases are often resolvable in one plane but not another. The femur and tibia may be bridged in axial slices (where they overlap) but clearly separated in coronal or sagittal views. Restricting examination to one plane forces the agent to declare futility when a different plane would have resolved the ambiguity.

The slice-examination-loop skill explicitly documents the multi-plane sweep as an OAMI step: render overview slices in all three planes before targeting specific locations.

---

## Tools Layer Completeness

### New modules

| Module | What it wraps | Why wrapped |
|---|---|---|
| `preprocess_tools.py` | `processing.preprocessing.median_filter_volume` | Median filter was in the processing layer but had no tool-layer interface. Agents couldn't call it directly without importing from `processing/`, which is supposed to be below the tool API surface. |
| `orientation_tools.py` | `processing.orientation.pca_orient`, `apply_rotation` | PCA orientation was in processing but had no tool-layer interface and no frame tracking. |
| `_plane_helpers.py` | Internal slice extraction helpers | Shared by `slice_renderer.py` and `slice_inspector.py` to avoid code duplication across the two rendering paths. |

### Extended modules

| Module | What was added |
|---|---|
| `segmentation_tools.py` | `create_sphere_markers`, `watershed_from_markers`, `run_mask_recipe` |
| `slice_renderer.py` | Sagittal and coronal rendering, mask overlay, point marker overlay |
| `slice_inspector.py` | Sagittal and coronal inspection, structured evidence output |

---

## Testing Philosophy

E2E workflow validation over unit tests. The real test for this pipeline is running the analyst agent on actual scan data and checking whether the measurements match published values.

Unit tests were written only for logic hard to verify through E2E:
- Sphere geometry edge cases (radius overflow, out-of-bounds clipping)
- Coordinate frame transform inversion correctness
- IIOC boundary detection algorithm on synthetic label data

The `validation/oa6-1rk-t1s-run/` directory holds the full E2E validation record: spawn command, prompt, results table, and reproducibility artifacts.

---

## OA6-1RK Validation Results

All 6 geometric indices passed the ±10% acceptance gate:

| Measurement | Result | Target | Delta | Gate |
|---|---:|---:|---:|---|
| Distal femur length | 2.2900 mm | 2.2900 mm | 0.00% | pass |
| Femur width | 3.5805 mm | 3.4800 mm | +2.89% | pass |
| Width/length ratio | 1.5635 | 1.5200 | +2.86% | pass |
| Tibia plate width | 2.9500 mm | 2.9500 mm | 0.00% | pass |
| IIOC height | 0.7455 mm | 0.7455 mm | 0.00% | pass |
| Plate H/W ratio | 0.2527 | 0.2530 | −0.11% | pass |

Widest delta: femur width at 2.89%, within the ±10% inter-reader tolerance from Tang et al.

**Significance:** This is the first time the automated pipeline has reproduced all six published values within the inter-reader agreement band. Prior implementations had 11–58% errors due to the landmark methodology issues documented in [measurement-design.md](./measurement-design.md).

---

## Deferred Items

Three items were explicitly deferred at the end of this implementation:

**1. PCA duplication across tool and processing layers.**  
`orientation_tools.py` wraps `processing.orientation.pca_orient`, but the two layers have overlapping PCA logic. Future cleanup should define a single PCA implementation, probably in processing, with the tool layer as a thin adapter. Deferred because both work correctly and unification is structural, not functional.

**2. Slice inspection shared primitive extraction.**  
`slice_renderer.py` and `slice_inspector.py` share conceptually similar slice extraction logic through `_plane_helpers.py`, but the two functions differ enough that further abstraction risks over-engineering the boundary. Revisit when a third consumer appears.

**3. Prompt-level executable tests for the slice-examination-loop skill.**  
The skill has documented behavior contracts but no prompt-level tests that run the agent against a real session. Prompt testing for skill behavior requires an actual kernel session with real scan data — the test infrastructure for this doesn't exist yet. Tracked as a capability gap in the prompt testing toolchain.

---

## Related Pages

- [overview.md](./overview.md) — agent architecture, session lifecycle, confidence gating
- [measurement-design.md](./measurement-design.md) — two measurement domains, OA6-1RK oracle
- [../jupyter-workbench/overview.md](../jupyter-workbench/overview.md) — runtime layer
- [../../open-questions/future-work.md](../../open-questions/future-work.md) — deferred items
