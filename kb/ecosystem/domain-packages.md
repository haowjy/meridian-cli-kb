# Domain Packages

Domain packages apply the Meridian ecosystem to a specific problem domain. Coverage here is intentionally light — these packages are operational, not infrastructure, and their internal details belong in their own docs.

For broader ecosystem context, see [ecosystem/overview.md](overview.md).

---

## microct-analysis

A mars prompt package for volumetric image analysis. The Python tools layer is **domain-agnostic** — domain knowledge enters through workflow definition files at runtime. Ships six agent profiles: `microct-analyst` (session-owning orchestrator), three specialist sub-agents (segmenter, landmarker, measurer), and two independent agents (workflow-creator, cleanup). The domain layer is pure: it owns segmentation, landmark placement, ROI definition, measurement execution, and the explain-then-apply interaction protocol, but delegates all kernel lifecycle, notebook persistence, and visualization to `jupyter-workbench` — its only runtime tool. `mouse-ct` is a required Python library dependency for segmentation stage drivers, not a runtime tool or architecture center.

**Key design:** Measurement requires two qualitatively different domains — femoral 3D surface measurements (Euclidean distances between surface landmarks) and tibial 2D slice measurements (slice count for IIOC height, oriented-slice distance for tibial width). Prior designs used a single Euclidean model and produced 11–58% errors vs published values. The watershed segmentation path uses small sphere markers (not flood-fill seeds) to correctly handle bridged-bone cases.

**Validated:** OA6-1RK (Tang et al. Table S3) — all 6 geometric indices passed the ±10% acceptance gate. Widest delta: femur width at 2.89%.

See [microct-analysis/overview.md](microct-analysis/overview.md) for agent architecture, mouse-ct integration, and workflow notes.  
See [microct-analysis/measurement-design.md](microct-analysis/measurement-design.md) for the two-domain measurement architecture, OA6-1RK oracle, and SOP-grounded processing decisions.  
See [microct-analysis/toolkit-implementation.md](microct-analysis/toolkit-implementation.md) for the generic tools layer design, watershed fix, 3DMedAgent pattern, and validation results.  
See [jupyter-workbench/overview.md](jupyter-workbench/overview.md) for the runtime layer.

---

## mouse-ct

A Python domain package (`mouse_ct/`) for mouse CT imaging analysis. Originally a standalone codebase with its own scene and visualization machinery, it was refactored as part of the jupyter-workbench build to adopt the workbench's execution and visualization substrate — separating the scene/plotter concerns, adopting transport-neutral events, stabilizing picker IDs, and rehoming QC and artifact conventions. After the refactor, mouse-ct is the pre-existing analysis codebase that `microct-analysis` agent skills operate on top of via the jupyter-workbench API. It is a Python package, not a mars prompt package.

---

## creative-writing-skills

A mars prompt package for producing long-form structured documents from many source inputs, with iterative revision that maintains coherence across the whole. Originally built as a standalone open-source project (~148 GitHub stars) before the Meridian ecosystem, it is the project that demonstrated the core multi-agent orchestration pattern — breaking a large writing task into coordinated agent subtasks — that Meridian later generalized. In the current ecosystem it is distributed as a mars prompt package, providing skills for document structure management, multi-source synthesis, and coherence-preserving revision. It has no runtime tool dependency beyond meridian-cli.

---

## Related

- [ecosystem/overview.md](overview.md) — full ecosystem map
- [jupyter-workbench/overview.md](jupyter-workbench/overview.md) — runtime layer for microct-analysis and mouse-ct
