# Jupyter Workbench + MicroCT Analysis — Overview

Status: Shipped (v0/v1 complete as of 2026-05-02)
Implementation: tech-lead spawn p3005

## What this is

A two-package system for persistent, notebook-backed analysis with live 3D visualization and bidirectional human-in-the-loop (HITL) interaction.

- **`jupyter-workbench`** — reusable Python library, agent-facing CLI, FastMCP adapter, and mars prompt package. Owns the kernel session, notebook lineage, snapshot observation, and visualization artifact conventions.
- **`microct-analysis`** — domain package for preclinical microCT workflows (mouse bone, OA models). Owns segmentation, landmarks, ROI, measurements, and explanation policy. Depends on `jupyter-workbench` public API only.

For full architecture, see [architecture.md](./architecture.md).
For session lifecycle, execution model, event capture, and visualization, see [runtime-model.md](./runtime-model.md).

---

## Two-Package Split: Why

Before this architecture, an earlier design had three components: a `jupyter-workbench` runtime, a `pyvista-cli` visualization runtime, and `microct-analysis`. The split-runtime model was eliminated.

**What changed:**

- `pyvista-cli` was eliminated entirely. PyVista + trame run **inside the Jupyter kernel**, not as a sibling process.
- `jupyter-workbench` became the only runtime tool — for compute, visualization, lineage, and cleanup.
- `microct-analysis` narrowed to a pure domain package: prompt skills and optional local helpers consuming the `jupyter-workbench` public API.

**Why this wins:**

| Prior framing | Revised framing | Why simpler |
|---|---|---|
| Two runtime CLI tools | One runtime tool: `jupyter-workbench` | Fewer interfaces, no split-brain state |
| Separate visualization daemon | Kernel hosts PyVista + trame | Removes daemon control plane |
| Viz verbs (`viz-show`, `viz-events`) | Visualization happens through `exec` + helpers | Reuses notebook execution substrate |
| Formal scene-spec schema between packages | Skill-taught patterns + durable artifact conventions | Less schema burden |
| Separate viz concurrency model | Single per-session workbench mutation lock | One serialization rule |

---

## Package Ownership

### `jupyter-workbench` owns

- Session manifests, kernel launch/reconnect/heartbeat, root/session resolution
- Append-only notebook writes and structured execution records
- Replay, derivation, and compaction for notebook lineage
- Visualization artifact persistence: browser URLs, events.jsonl, screenshots, scene metadata
- Typed DTO-returning core services: `SessionService`, `ExecutionService`, `SnapshotService`, `LineageService`
- FastMCP wrappers that serialize the same DTO field semantics as the CLI/library API
- CLI verbs: `open`, `exec`, `markdown`, `snapshot`, `status`, `list`, `close`, `replay`, `lineage`, `derive`, `compact`
- Skills: session-management, notebook-lineage, compaction-cleanup, pyvista-interactive

### `jupyter-workbench` must not own

- MicroCT anatomy semantics
- Segmentation, landmark, or measurement policy
- Domain-level interpretation of picks and interaction events

### `microct-analysis` owns

- Segmentation, landmarks, ROI boxing/squaring, measurements, structure identification
- Scene semantics: bone color palettes, anchors, QC views
- Plain-language explanation policy ("explain-then-apply" protocol)
- Translation from user feedback into domain corrections
- Workflow split between expensive interactive analysis and cheap standalone cleanup
- Domain event interpretation (mapping generic `pick` → `component_picked: femur_fragment_3`)

### `microct-analysis` must not own

- Kernel lifecycle
- Notebook persistence or replay
- Direct imports of `jupyter_workbench.adapters.*` (internal implementation detail)

---

## Install Contract

```bash
uv add jupyter-workbench            # library dependency
uv tool install jupyter-workbench   # direct agent CLI
uv tool install 'jupyter-workbench[mcp]'  # MCP server support
uvx jupyter-workbench <verb>        # ephemeral, no install needed
jupyter-workbench-mcp               # FastMCP server entry point
```

`microct-analysis` is a mars prompt package with no binary install. Its bootstrap fails loudly if `jupyter-workbench` is not available.

---

## First Wedge

The initial `microct-analysis` target is preclinical microCT imaging: mouse bone models, OA research, morphometric measurements. See strategy docs for market context:
- `strategy/archive/legacy-flat-docs/2026-04-29/first-wedge-microct.md`
- `strategy/product/research-output-environment.md`

---

## Key Decisions (Summary)

| Decision | Choice | Rationale |
|---|---|---|
| Product split | Two packages, no `pyvista-cli` | Eliminates extra runtime boundary while preserving domain separation |
| Kernel substrate | `jupyter_client` inside `jupyter-workbench` | Best fit for persistent kernels, structured outputs, real `.ipynb` fidelity |
| Viz stack | PyVista + trame inside kernel session | One long-lived process; no daemon; one concurrency model |
| Agent interface | Single CLI-first: `jupyter-workbench` only | Reuses one session contract for compute, visualization, lineage, cleanup |
| Recovery | Explicit replay + derivation/compaction | Makes failure recovery observable; preserves provenance |
| Governing pattern | Ports and Adapters + SOLID | CLI/MCP/library parity automatic; core testable; new adapters by addition |

See [architecture.md](./architecture.md) for the decision log and rationale behind each choice.
