# Thermo-Nuclear Audit: Method and Rejected Alternatives

Full-structure audit of meridian-cli (issue #389, 2026-07). Aimed to
eliminate bug classes by construction — make bugs structurally unable to
happen, with performance regressions counted as a bug class. This page
records the verification method that worked and the alternatives the
adversarial panel rejected with reasons.

## Verification Method

The audit used a two-source adversarial panel for every code finding
before it could enter the deliverable.

**Structure.** 17 finder lanes decomposed from the qi/knowledge layer
tree (9 subsystem, 3 peer-benchmark, 4 cross-cutting, 1 documentation).
76 raw code findings were grouped into 10 verification clusters (A-J),
each verified independently by two sources on different model families
and harnesses:

1. A **Fable refuter** (native lane) — opens every cited file:line, tries
   to disprove the finding.
2. A **sol reviewer** spawned via `meridian spawn -a reviewer --skills
   thermo-nuclear-review --bg` — bridged through the subagent protocol,
   reads the same files, same goal.

Two harnesses, two model families, same cited evidence. Agreement produces
the verdict. Disagreement escalates to orchestrator adjudication — the
orchestrator decides, never averages.

**What the panel caught.** Five findings were dropped or merged after panel
disagreement:

- **[50] shared `read_json_object` helper**: refuted — the four "copies"
  implement different contracts (SQLite column values, missing-vs-unreadable
  taxonomy); only two shallow duplicates remain, and a cross-layer API would
  be a leaky abstraction.
- **[19] drain-coordinator forwarding mixin**: refuted — the `DrainCoordinator`
  protocol + typed `DrainPlan.coordinator` field already enforce shape
  statically; a mixin of ten trivial forwards hides explicit composition.
- **[40] `resume_fidelity` capability field**: refuted — Vercel has no static
  recovery-fidelity metadata (its docs explicitly say there is no static
  capabilities object). The field would have zero consumers and conflates
  completed-session resume with in-flight-turn recovery.
- **[31] "all five adapters leak"**: refuted as framed — meridian's
  disk-recorded process scopes + reaper cleanup + Linux PDEATHSIG already
  delete the claimed class. Only the narrowed residue (SpawnManager
  registration window + stdio children without scope sidecars) was kept.
- **[29] Mermaid JS validation tier**: reclassified — the bundle was never
  committed or shipped, so `detect_tier()` always selects Python. Dead code,
  not a performance bug.

Several findings were **reproduced with runtime probes** by the sol
reviewer, including four lost-update failures in the spawn/work/hook
stores, lock split-brain on POSIX, nonterminal finalize acceptance, and
cross-harness event misclassification.

**Why it works.** The method catches both false claims (the finding says
something is broken that is not) and wrong remedies (the finding is right
but the proposed fix is worse than the disease). A single-source review
would have shipped [50] and [40] — both were plausible and well-cited but
wrong at a level that requires opening the surrounding code, not just the
cited lines.

## Rejected Alternatives

Each alternative was proposed during the audit and rejected by the panel
with specific evidence. These are design decisions — they explain what NOT
to do and why.

### SpawnOwnership capability object

**Proposal**: a process-local capability object that grants exclusive write
access to a spawn's state, passed from publish to runner.

**Why rejected**: the publish/run boundary is a process split (background
spawns persist a `BackgroundWorkerLaunchRequest` to disk, then a separate
process picks it up). A capability object cannot span that split. The real
fix is collapsing the unlocked owner-write tier into the locked mutation
seam — once every mutation goes through `write_state_locked`, "ownership"
stops being a write-safety concept.

### Exactly-once reaper kill gating

**Proposal**: gate process kills strictly on CAS victory so cleanup is
exactly-once.

**Why rejected**: exactly-once breaks crash-only recovery. If the reaper
crashes after the CAS but before termination, the orphaned scopes are
never cleaned up because reconciliation skips terminal rows. The correct
pattern is at-least-once: persist a recoverable cleanup claim under
`state.lock`, then perform idempotent birth-validated termination whenever
the post-CAS snapshot is terminal and a recorded scope still exists.

### Wholesale TypeAdapter config replacement

**Proposal**: replace the settings normalizers with a single Pydantic
TypeAdapter wholesale.

**Why rejected**: the normalizers implement behavior, not just parsing.
Sparse-layer merge via `model_fields_set`, unknown-key warnings,
`work.artifacts` coercions, and Pi-path coercions are load-bearing
logic. Wholesale replacement would silently drop those behaviors.
Separate the source decoders from the validation contracts and centralize
only shared semantic constraints.

### Shared `read_json_object` helper

**Proposal**: unify the four JSON-reading call sites into one canonical
helper.

**Why rejected**: the four "copies" implement different contracts. One
parses SQLite column values (no file, no `OSError`). One needs a
missing-vs-unreadable taxonomy that `dict | None` collapses. Only two are
shallow duplicates. A cross-layer API would be a leaky abstraction.

### ruff TID251 for the raw-write ban

**Proposal**: use ruff's TID251 rule to enforce the
atomic-write-only contract (ban `Path.write_text` etc.).

**Why rejected**: TID251 matches import paths, not method calls on
inferred types. It cannot flag `some_path.write_text()` because it does
not know `some_path` is a `Path`. The enforcement mechanism is an
AST-based conformance test with a documented allowlist.

### Strict rejection at raw harness-event parsing

**Proposal**: hard-fail on unknown upstream event types during stream
parsing (Vercel's discriminated-union compile-time guard model).

**Why rejected**: meridian ingests unpinned third-party streams. Harness
vendors add event types across minor releases. Hard-failing on unknown
events would break every harness upgrade. The correct pattern: keep an
open `RawHarnessEvent` transport envelope and normalize into a closed
semantic union. Unknown events pass through the raw layer; only the
semantic layer enforces the closed vocabulary.

## Peer-Benchmark Verdicts

The audit validated three architectural strengths against peer codebases
and confirmed they should be defended, not simplified toward peer patterns.

### Defended: descendant-aware completion

Meridian's resident/Pi completion holds until every transitive descendant
`state.json` is terminal. The reaper classifies outcomes
(`orphan_finalization`/`orphan_run`/`timed_out`) rather than kill-and-rebuild.
Peer contrast: omnigent has no analog — `subagents` is a flat capability
boolean; sub-agent dispatch is fire-and-forget.

### Defended: journaled permission broker

`permission_broker.py` persists every approval as an append-only sequenced
journal with pending/resolved/failed/cancelled transitions. Crash recovery is
independent of the orchestrator process. Peer contrast: Vercel's
`pendingToolApprovals` is an in-memory Map whose durability depends on
an external Workflow DevKit.

### Defended: transitive drain evidence

`ReconciledDescendantEvidence` is shared by resident and Pi paths — one
canonical helper, not per-harness forks. Peer contrast: no peer in the
corpus implements descendant-aware completion at all.

### Dropped steal: Vercel recovery classification

The `resume_fidelity` capability field proposed from Vercel's pattern was
refuted. Vercel has no static recovery-fidelity metadata. The field would
have zero consumers and conflates completed-session resume (which meridian
already models via `supports_session_resume` / `supports_session_fork`)
with in-flight-turn recovery (which no harness currently supports).

## Execution Outcome: Concurrency by Construction (PR #422)

The audit's concurrency findings drove PR #422 — four waves of structural
fixes that made the lost-update, split-brain, kill-mid-run, and torn-write
bug classes unwritable by construction.

**What shipped:**
- One parameterized lock primitive (timeout, shared mode, reentrancy,
  fork-safety) over stable never-unlinked lock inodes
- One atomic-replace context manager in a dependency-neutral platform layer
- Mutate-under-lock seams across all stores: spawn records, archived
  spawns, work items, hook intervals, scope projections, autosync
  transactions
- Reaper finalize-first with crash-recoverable at-least-once cleanup claims
- Autosync single-transaction execution with canonical lock path
- Repo-wide AST conformance guard rejecting raw state writes at CI

**Hardening found by gate reviews:** project pruning destroying live runtime
roots (project-lifetime shared/exclusive gate); published-spawn deletion
racing locked writers (one locked deletion seam); fork children inheriting
and prolonging parent locks (process-wide registry, fork-aware); atomic
replace flipping user files to 0600 (explicit permission policy).

**Rejected alternatives confirmed:** exactly-once kill gating
(breaks crash-only recovery), SpawnOwnership capability object (cannot span
process split), ruff TID251 (cannot match method calls on inferred types),
revision-CAS without re-read-and-reapply, shared `read_json_object` helper
(different contracts).

Five review gates, every gate reproduced at least one real bug before the
fixes shipped. 1563 tests passed / pyright 0 errors at final gate.

See [decisions/state.md](../decisions/state.md) for the durable decision
records.

## Provenance

work:thermo-nuclear-audit. Audit PR #425 (closes #389). Verification
corpus: `findings/` (17 lane reports + `merged.json`), `verify/`
(per-cluster dossiers A-J + Fable refute notes + sol reviewer reports).
Concurrency-by-construction execution: PR #422 (work:concurrency-by-construction).
