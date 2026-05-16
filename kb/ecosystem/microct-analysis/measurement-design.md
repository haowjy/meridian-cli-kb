# MicroCT Analysis — Measurement Design

Status: Design approved (2026-05-03)  
Ground truth source: Tang et al., Biology 15(3):262 (doi:10.3390/biology15030262)  
Package architecture: [overview.md](./overview.md)

---

## 1. Root Cause of Prior Measurement Errors

The original pipeline produced measurements 11–58% off from published values for OA6-1RK because landmark identification did not match the paper's actual methodology.

| Measurement | Published | Pipeline | Error | Root cause |
|---|---|---|---|---|
| Distal femoral length | 2.29 mm | 3.43 mm | +50% | Used label extent instead of surface landmark distance |
| Distal femoral width | 3.48 mm | 3.87 mm | +11% | Label bbox instead of surface edge-to-edge |
| Width/length ratio | 1.520 | 1.13 | −26% | Both components wrong, errors compound |
| IIOC height | 0.7455 mm | 1.0 mm | +34% | Distance between 3D points instead of slice count |

**The unifying problem:** landmarks were label-derived approximations (centroids, bounding box extremes) instead of anatomically defined surface features. Wrong landmarks → wrong distances.

---

## 2. Two Measurement Domains

The paper uses qualitatively different methods for femoral vs tibial parameters. A single "place landmarks, compute Euclidean distance" model cannot reproduce both. This is an architectural requirement, not a parameter tweak.

### Domain 1: femoral_3d_surface

**Tool in paper:** Amira Ruler tool on 3D rendered surface model.  
**Method:** Operator clicks two surface points on the 3D bone model; software computes straight-line Euclidean distance in physical coordinates.

Characteristics:
- Landmarks are specific anatomical surface features on the segmented bone mesh.
- Distances are 3D Euclidean (length) or frontal-plane projected (width — see §2.1).
- No orientation correction required. 3D distances are rotation-invariant.

**Femoral length:** Full 3D Euclidean distance between `intercondylar_groove_midpoint` and `intercondylar_notch`. The two points are at different anterior-posterior depths, so the full 3D distance is correct.

**Femoral width:** **Frontal-projected distance**, not 3D Euclidean. The paper measures "on the front view of µCT images." Width is the medial-lateral span between condylar edge landmarks, excluding the anterior-posterior depth component:

```
width = abs(lateral_edge.x − medial_edge.x) × spacing_x
```

**Why surface features, not label statistics:**
- The groove midpoint is a surface concavity; the label centroid is ~1 mm away.
- The intercondylar notch is between the condyles; the label's inferior Z-extreme is the bottom of the condyles.
- Published femoral length (2.29 mm) vs label Z-extent (~3.4 mm): the label includes the femoral shaft proximal to the groove, which is not part of the distal femur measurement.

**Femoral landmark identification:**

| Landmark | Geometric identification |
|---|---|
| `intercondylar_groove_midpoint` | Saddle point at the superior end of the patellar groove — local ML-minimum that is a SI-maximum on the anterior distal femoral surface |
| `intercondylar_notch` | Deepest point within the intercondylar fossa (posterior/inferior surface), most proximal point in the inter-condylar space |
| `lateral_condylar_edge` | Outermost lateral surface point of the distal femur, **including osteophytes**, projected onto the frontal plane |
| `medial_condylar_edge` | Outermost medial surface point, **including osteophytes**, projected onto the frontal plane |

**Osteophytes are included deliberately.** The paper explicitly states the width represents "osteophyte formation" — the condylar edge landmarks are the outermost points of bone including pathological outgrowths, not the anatomical condylar margin. This makes femoral width an OA indicator.

### Domain 2: tibial_2d_slice

**Tool in paper:** Amira Transform Editor + Ortho Slice + Ruler tool.  
**Method:** (1) Rotate the µCT dataset until the tibia is frontal-plane aligned. (2) Select the specific oriented slice. (3) Measure distances or count slices on the 2D cross-section.

Characteristics:
- Orientation correction must be applied **before** any tibial measurement.
- Specific slice selection criteria determine which slice to measure.
- IIOC height is a **slice count**, not a 3D distance.

**IIOC height is strictly a slice count.** From the paper methods: "The maximum height between growth plate and tibia articular surface was calculated by counting the µCT slices between the most proximal appearance of the articular surface and the most proximal appearance of the growth plate. Each slice is 10.5 µm."

Verified against Table S1: OA6-1RK "BV-ArticularSurface-GrowthPlate" ROI has Dim-Z = 71 slices. 71 × 0.0105 mm = 0.7455 mm, which matches the Table S3 "Max Height II-OC" value exactly. All published IIOC heights in S3 are exact multiples of 0.0105 mm.

**Tibial width** is measured on a specific oriented frontal slice at growth plate level, not as a 3D bounding-box width.

---

## 3. OA6-1RK as Acceptance Oracle

OA6-1RK is the sample with known DICOM data and published reference values from Tang et al. Table S3. It is the acceptance specification for the pipeline.

**Published reference values (OA6-1RK):**

| Measurement | Published value | Acceptable range | Tolerance basis |
|---|---|---|---|
| Distal femoral length | 2.29 mm | 2.06–2.52 mm | ±10% (inter-reader) |
| Distal femoral width | 3.48 mm | 3.13–3.83 mm | ±10% |
| Width/length ratio | 1.520 | 1.37–1.67 | ±10% |
| Tibial IIOC height | 0.7455 mm | 0.714–0.777 mm | ±3 slices (0.032 mm) |
| Tibial width | 2.95 mm | 2.66–3.25 mm | ±10% |
| IIOC H/W ratio | 0.253 | 0.228–0.278 | ±10% |

**Tolerance rationale:** The paper's two trained readers differ by 8–9% on ratios (1.520 vs 1.404 for femoral; 0.253 vs 0.232 for IIOC). The pipeline cannot reasonably match tighter than human-human agreement. IIOC height gets ±3 slices rather than ±10% because boundary detection has less subjective variation than surface landmark placement, and 3 slices is consistent with algorithmic boundary uncertainty.

**What OA distinguishes from normal:**

| Metric | Normal (max) | OA threshold |
|---|---|---|
| Femoral width/length ratio | < 1.28 | > 1.30 |
| Femoral length | 2.18–2.49 mm (stable, unchanged by OA) | — |
| Tibial IIOC H/W ratio | 0.289–0.318 | < 0.282 (subchondral collapse) |

Femoral length is stable across groups (mean 2.37 normal vs 2.36 MMS 8wk) and serves as the normalizing denominator. The ratio is the OA indicator, not the absolute length.

---

## 4. PCA-Based Tibia Orientation (Design Extrapolation)

### What the paper does

The paper uses Amira's Transform Editor for manual rotation: the operator visually rotates the µCT dataset until the tibia is frontal-plane aligned. Table S2 records the per-sample x/y/z offset coordinates used. The paper does **not** describe an algorithmic orientation procedure. This is a manual visual judgment.

**The pipeline cannot claim to reproduce the paper's exact orientation.** PCA-based alignment is an engineering approximation that must be separately validated.

### The PCA algorithm

0. **Center first (from Amira SOP):** Compute the bounding box of the tibia label. Apply `translate = −1 × (LL + UR) / 2` to center on the origin before any rotation. The SOP performs this step in Amira before rotating. Centering ensures rotation is applied around the anatomical center, not an arbitrary corner.

1. **Extract tibia principal axis:** PCA on thresholded tibia label voxel positions (on the centered volume). First PC = long axis.

2. **Align long axis to Z:** Apply a rotation matrix mapping the first PC to the Z-axis (superior-inferior).

3. **Determine frontal roll:** Use the second PC (widest cross-section direction) to approximate the medial-lateral axis. Align to X-axis.

4. **Apply rigid transform (two volumes separately):**
   - Label mask: `order=0` (nearest-neighbor). Linear interpolation on binary labels creates mixed voxels that corrupt boundary detection.
   - Intensity volume: `order=1` (linear). Original intensity data is needed for growth plate detection (270/220 threshold ratio).
   - **Preserve voxel size.** Do not rescale spacing — slice count × 0.0105 mm must remain valid.

5. **Validate against Table S2:** For OA6-1RK, the oriented tibia must produce IIOC height = 71 ± 3 slices and tibial width = 2.95 ± 0.30 mm.

### Fallback: user-assisted orientation

If the automated alignment produces measurements outside acceptance tolerance, the pipeline escalates: display the oriented tibia in the PyVista scene, ask the user to confirm or adjust frontal alignment, and record any correction as a run override. This mirrors the paper's actual methodology.

### Why PCA may fail

PCA aligns by mass distribution. If the tibia has asymmetric osteophyte growth, the PCA axes may be rotated relative to the true anatomical axes. The fallback is always user-assisted.

---

## 5. Growth Plate Separation (from Amira SOP)

The ROI for trabecular morphometry is not a simple bounding box. It is produced by a multi-step procedure from the Amira SOP (`Amira Micro-CT Analysis.docx`), extracted 2026-05-04.

### Procedure

1. **Morphological closing recipe:** Run the Amira closing recipe on the tibia label to produce `Result.closing*`.
2. **Shrink × 3 (all slices):** Select the interior of the bone, apply Shrink → Replace three times.
3. **Lock Exterior:** Freeze the exterior voxels (the cortical shell).
4. **Threshold LB=1000–1500 on the interior:** Select masked voxels → classify as **Subchondral bone**.
5. **Residual interior → Marrow.**
6. **Crop ~20–25 slices starting at beginning of growth plates.**
7. **Create total bone = subchondral bone + marrow.**
8. Run Material Statistics on both.

**Key insight:** The shrink→lock→threshold pattern respects the cortical boundary algorithmically. Plain thresholding at a single value bleeds across the cortical shell. The recipe produces a closed cortical mask; the manual steps classify the interior as subchondral vs marrow using the lower intensity range (LB=1000–1500 captures low-density subchondral bone missed at the bone threshold LB=2500).

The medial/lateral compartment split is "determined visually through the center of the proximal tibia." There is no algorithmic rule — the pipeline must prompt the user for this judgment.

**Pipeline implementation:** `morphology.isolate_trabecular_roi` implements this procedure, not plain thresholding. `morphology.crop_growth_plate_region` handles the crop.

---

## 6. Amira LB vs SCANCO Threshold Distinction

The SOP uses Amira Lower Bound (LB) values to control the visual workflow in Amira software. SCANCO thresholds are the quantitative reference for published values. They are related but not identically calibrated.

| Amira LB | Purpose in SOP | SCANCO equivalent | Use in pipeline |
|---|---|---|---|
| LB=2500 | Volume rendering, bone/soft-tissue mask | 220 | Segmentation and most quantitative analysis |
| LB=3500 | 3D display threshold | 270 | Subchondral/cortical analysis |
| LB=320 (implied) | 3D surface segmentation | 320 | 3D surface rendering |
| LB=1000–1500 | Subchondral vs marrow separation | context-dependent | Interior classification (not reported metrics) |

**All pipeline threshold values and published acceptance numbers use SCANCO thresholds (220, 270, 320).** Amira LB values appear only in operator documentation; they are not used in the pipeline.

---

## 7. IIOC Boundary Detection Algorithm

Both boundaries operate on the **oriented tibia segmentation label volume** (binary: tibia voxels vs background), not on raw voxels.

**Articular surface boundary (superior):**
1. Start at the most superior Z-slice.
2. Scan inferiorly. The boundary is the first slice where the tibia label occupies ≥ 20% of the maximum per-slice tibia area in the volume (excludes noise/partial-volume edge contacts).
3. If no slice reaches 20%, use the first slice with any tibia voxels but flag confidence as `low`.

**Growth plate boundary (inferior):**

Source volume: oriented **original intensity volume**, masked by the oriented tibia label. The intensity values are needed to compute the 270/220 ratio — the binary label alone cannot distinguish mineralized bone from cartilage.

1. Start from the articular surface boundary.
2. Scan inferiorly. On each slice, compute the bone-fill ratio: (voxels above threshold 270) / (voxels above threshold 220) within the tibia label mask on that slice.
3. The growth plate boundary is the first slice where the ratio drops below 50% after having been above 50% for at least 5 consecutive slices.
4. Confirmatory check: on the identified slice, verify a continuous low-density band spans >50% of tibia width. If absent, flag `medium` confidence but keep the ratio-based boundary.
5. Scan from the articular surface distally to prevent finding the cortical shell transition instead of the growth plate.

**Validation:** OA6-1RK must produce 71 slices between boundaries (±3 slices).

---

## 8. Key Measurement Decisions

| Decision | Choice | Rationale |
|---|---|---|
| ML1: Two measurement domains | Required | Paper uses qualitatively different methods for femoral vs tibial — a single Euclidean model cannot reproduce both |
| ML2: Landmarks are surface features | Required | Published values prove centroid/bbox landmarks produce 11–58% errors |
| ML3: IIOC height is a slice count | Required | Verified against S1 ROI dimensions — every published value is an exact multiple of 0.0105 |
| ML4: Metric-specific tolerances | Ratios ±10%, IIOC ±3 slices | ±10% from inter-reader variability (8–9% paper readers). IIOC ±3 slices from boundary detection uncertainty |
| ML5: Orientation before tibial measurements | Required | Paper uses Transform Editor before Ortho Slice; tibial measurements are undefined without frontal alignment |
| ML6: Include osteophytes in femoral width | Required | Paper explicitly states width "representing osteophyte" |
| ML7: Femoral width is frontal-projected | Required | Paper measures "on the front view" — excludes AP depth component |
| ML8: PCA orientation is design extrapolation | Acknowledged | Paper uses manual Transform Editor; PCA requires separate validation against Table S2 |
| ML9: Nearest-neighbor resampling for label masks | Required | Linear interpolation creates mixed voxels on binary labels, corrupting boundary detection |

---

## Related Pages

- [overview.md](./overview.md) — agent architecture, mouse-ct positioning, workflow system
- [jupyter-workbench/overview.md](../jupyter-workbench/overview.md) — runtime layer
