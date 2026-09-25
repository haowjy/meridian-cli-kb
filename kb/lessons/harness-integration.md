# lessons/harness-integration — Lessons from Integrating Multiple AI Harnesses

These are the non-obvious discoveries from integrating Claude, Codex, and OpenCode into a single harness-agnostic coordination layer. Each lesson came from a real integration challenge.

## Why PTY Capture for Claude Primary Launch

**The constraint:** Claude's interactive TUI emits its session ID to the terminal on startup. There's no flag to get the session ID programmatically, and it doesn't emit structured JSON on stdout during interactive mode.

**The naive approach:** Pipe stdout and parse for the session ID. This fails because the TUI is designed to talk to a terminal. Interactive behavior, colors, and in-place rendering require a TTY. Piping stdout gets garbled or suppressed output.

**The solution:** Launch Claude under a PTY (pseudo-terminal). The PTY convinces Claude it's talking to a real terminal. Meridian reads the PTY output, extracts the session ID from the startup banner, and then hands the PTY to the user's terminal so the interactive session proceeds normally.

**The lesson:** When a tool is designed for humans, observation requires meeting it on its terms. PTY capture is the minimum machinery needed to observe session ID without reimplementing Claude's TUI. It's explicitly limited to that extraction use case — not used as a general control mechanism.

**Where this lives:** `launch/process/pty_launcher.py`, `harness/adapters/claude.py:detect_primary_session_id()`

**Identity no longer depends on this.** Claude accepts `--session-id <uuid>`, so
Meridian now assigns the ID and binds it before exec. PTY-observed IDs can only
confirm it or conflict with it ([native session binding](../architecture/native-session-binding.md)).

---

## Why Managed Primary Attach for Codex

**Codex's architecture:** Codex is not a traditional CLI tool that you invoke and wait for. It's an app-server model: a persistent HTTP/WebSocket server that manages agent state, with a TUI that connects to it as a client.

**The constraint:** Meridian needs to track the session, inject context, and observe outputs. It can't do this by wrapping a black-box process.

**The solution:** Managed primary attach. Meridian:
1. Starts the Codex app-server
2. Gets an observer connection (WebSocket)
3. Attaches the TUI to the server
4. Streams events from the observer connection to `history.jsonl`

The user sees the normal TUI. Meridian sees all events.

**The lesson:** Different harnesses have different architectural models. Claude is "invoke and observe output." Codex is "connect to a running server." The adapter layer must accommodate both, and the connection machinery (`harness/connections/`) enables this without exposing the difference to the rest of the system.

**Where this lives:** `launch/process/primary_attach.py`, `harness/connections/codex_ws.py`

---

## Why Inline Content for Codex and OpenCode

**The problem:** Claude has a dedicated `--append-system-prompt-file` flag for system prompt injection. Codex and OpenCode don't.

**The first attempt:** Native file injection flags (e.g., `--file`). OpenCode had a `--file` flag that could inject reference content. We used it.

**Why it failed:** Native file injection is harness-specific. The injection semantics differ across harnesses — some treat it as a system message, some as a user message, some as context. Managing the injection contract per-harness in the composition layer created adapter-specific branching that was hard to reason about.

**The solution:** Always-inline reference routing. For Codex and OpenCode, reference content is rendered inline in the prompt as markdown blocks. The prompt is the universal channel — every harness reads it. `supports_native_file_injection` is `False` for all current harnesses; the `--file` injection path was removed.

**The tradeoff:** Inline content inflates the prompt size. For large reference files this can push toward token limits. The `MAX_INITIAL_PROMPT_BYTES` limit (10 MiB) guards against the worst cases, and `PromptTooLargeError` is surfaced to the caller.

**The lesson:** When harness capabilities diverge, prefer the universal channel (inline prompt) over per-harness native channels. Native channels add adapter-specific complexity without proportional benefit unless the native channel provides something the universal channel fundamentally cannot.

---

## The Semantic IR + Projection Pattern

**The problem it solved:** Each harness has different channels for different content types. Claude takes a system prompt separately from the user turn. Codex expects everything flattened. OpenCode has its own structure. How do you write content composition once and project to any harness?

**The first approach:** Compose for each harness. This meant the prompt assembly code knew about Claude, Codex, and OpenCode. Adding a harness meant editing composition code.

**The solution:** Two-stage composition:

1. **Semantic IR** (`ComposedLaunchContent`) — harness-agnostic. Contains: `SYSTEM_INSTRUCTION` (skills, profile, instructions), `TASK_CONTEXT` (reference files, project context), `USER_TASK_PROMPT` (the user's prompt).

2. **Adapter projection** (`project_content()` on each harness adapter) — maps IR to harness channels. Claude puts `SYSTEM_INSTRUCTION` in `--append-system-prompt-file` and `TASK_CONTEXT + USER_TASK_PROMPT` in the user turn. Codex flattens everything inline. OpenCode uses the same inline flattening.

**The lesson:** Representing composed content as an intermediate data type (IR) decouples the "what should the agent know" question from the "how does this harness consume it" question. New harness = implement `project_content()`. No changes to composition code.

**Where this lives:** `launch/composition.py` (IR types), `harness/adapter.py:project_content()` (projection protocol)

---

## Reference Routing Evolution

**Original design:** Reference files were injected through the harness's native file mechanism when available, inline otherwise. The `reference_input_mode` capability flag controlled routing.

**The problem:** Native injection semantics were inconsistent — Claude's `--file` had different behavior from OpenCode's `--file`. The flag led to code that silently changed behavior based on harness, and bugs that only appeared with specific harnesses.

**The decision (D-ref-routing):** Remove `reference_input_mode` entirely. Always route references as inline content. Each adapter's `project_content()` decides per-item whether to `"inline"` or `"omit"` (empty-body files are omitted to avoid noise). No more native injection path.

**The lesson:** A capability flag that controls behavior with correctness implications is dangerous. It invites "just use native mode" shortcuts that bypass the universal invariants. The flag was removed and the contract simplified to: references go inline, always.

---

## PTY Thread Safety: Signal Handlers and Worker Threads

**The bug:** When PTY capture is running in a worker thread (for async launch integration), SIGINT arrives at the main thread — as it should, per POSIX. But the signal handler was set up from the worker thread's context.

**The constraint:** Python only allows signal handlers to be installed in the main thread. Installing one from a worker thread raises `ValueError`.

**The lesson learned the hard way:** PTY lifecycle management (signal setup, terminal restoration) must happen in the main thread. The PTY capture worker can run in a thread, but setup and teardown must be coordinated with the main thread. This adds boilerplate but is non-negotiable.

**Where this manifests:** `launch/signals.py` (signal coordination), `launch/process/pty_launcher.py` (PTY launch with main-thread signal setup)

---


## Why Extension Injection for Pi (Managed-Bash and Spawn Watch)

**The problem:** Pi's JSONL RPC event stream doesn't surface background job completion or session quiescence. No `job.finished` or `session.idle` event exists. Meridian needs to know when tracked child work completes before declaring a spawn done — and Pi's protocol doesn't provide it.

**The solution:** Meridian injects TypeScript extensions into Pi via `-e <path>` flags. These extensions run inside Pi's process, access Pi's bash execution context, and write durable coordination state. Python watches that disk state to decide quiescence.

Two managed extensions ship as package data:
- **`managed-bash`** — overrides Pi's bash builtin. Registers `bash` tool (with `background?: boolean` + `timeout_min?: 1-59` parameters) and `bash_manage` ops tool. Returns immediately with `{bash_id, status: "backgrounded"}` for tracked bg transitions; blocks for synchronous calls.
- **`meridian-spawn-watch`** — observes spawn records for the current Pi session, dispatches implicit-wait completion notifications (`sendMessage({triggerTurn: true})`) when tracked work terminates, writes a `last-notification.json` marker, and provides the `/spawn` family of slash commands (renamed from `/mspawn` — no compatibility alias).

**Why disk-state observation:** `managed-bash` writes `pi-bash/<spawn-id>/bash-records.json`; `meridian-spawn-watch` writes `pi-bash/<spawn-id>/last-notification.json`; Meridian's spawn store writes `runtime_root/spawns/<child>/state.json`. Python-side `PiDiskWatcher` wakes `PiQuiescenceTracker` on those changes. Disk state is the canonical signal; env-var correlation (`MERIDIAN_PI_BASH_ID` → `originating_bash_id`) bridges bash and spawn observations without command-string parsing.

**The lesson:** When a harness's native protocol doesn't expose enough structure for quiescence detection, extension injection lets Meridian add a code path inside the harness process rather than working around it. The tradeoff is coupling to Pi's extension API: if Pi changes its extension loading mechanism, the extensions may need updates. The compatibility probe (`pi --help` must advertise `--no-extensions` and `-e`) gates the harness before any spawn.

**Where this lives:** `src/meridian/pi_runtime/extensions/` (managed-bash + meridian-spawn-watch), `src/meridian/lib/harness/connections/pi_rpc.py`, `src/meridian/lib/streaming/pi_drain.py`, `src/meridian/lib/streaming/disk_watcher.py`, `src/meridian/lib/streaming/pi_quiescence.py`

---

## Pi Runtime Resolution vs Bundled Runtime

**Prior harness pattern:** Claude, Codex, and OpenCode are launched by name via PATH (`claude`, `codex`, `opencode`). No pre-launch probe, no compatibility check.

**Why Pi differs:** Pi's RPC mode and extension loading surface (`--mode rpc`, `-e`, `--no-extensions`, `--session-dir`) are required for Meridian to function correctly. If the installed `pi` binary doesn't support these flags, the spawn fails mid-run with an unhelpful error — or worse, silently degrades.

**The solution:** Probe-before-launch via `PiRuntimeResolver`:
1. Resolve binary: `MERIDIAN_PI_BINARY` override, then `pi` from PATH.
2. Run `pi --version` — must exit 0.
3. Run `pi --help` — must advertise required flags (`--mode rpc`, `-e`, `--session-dir`, `--no-extensions`, `--append-system-prompt`, `--session`, `--fork`).
4. Fail fast with actionable guidance if probe fails.

**The lesson:** For harnesses where the required CLI surface is non-trivial, probe-before-launch gives a clear pre-launch error rather than a cryptic mid-run failure. The probe cost (two fast subprocesses) is negligible compared to the spawn itself. This pattern should be adopted for any future harness that requires specific CLI flags or modes that may not be present in all versions.

**Where this lives:** `harness/pi_runtime_resolver.py`, `harness/pi.py:_resolve_binary()`

---

## Resident Completion Needs a Harness Control Seam

**The bug:** Codex/OpenCode spawns could finish a turn while Meridian-tracked
`--bg` child work was still active. Treating the successful turn frame as finalization
closed the parent early; if a child was still launching or retrying a managed backend,
the backend could survive as an orphan.

**The tempting fix:** Teach the generic drain loop to keep every successful terminal
frame open until descendants drain. That would have mixed child-tree policy into the
plain streaming path and made harness terminal frames ambiguous everywhere.

**The solution:** Keep the generic path plain and add a narrow resident seam.
Connections that can keep a backend alive expose `resident_backend`; `SpawnManager`
selects `ResidentDrainCoordinator` only for those connections. A successful turn becomes
a turn boundary while descendants are active. The coordinator consumes file signals
(`meridian spawn done` / `rearm`), starts follow-up turns through
`ResidentBackendControl.begin_followup_turn()`, and uses a deadline backstop that
finalizes the parent `timed_out` while cancelling active descendants through the normal
cancel pipeline.

**The lesson:** Completion authority is not always the same thing as "the harness
emitted a terminal-looking frame." When a harness can stay resident, model the extra
control as a capability seam and keep absence of that seam as the ordinary path.
Do not create a no-op coordinator just to make the generic loop look uniform.

**Where this lives:** `src/meridian/lib/streaming/drain_coordinator.py`,
`src/meridian/lib/streaming/resident_drain.py`,
`src/meridian/lib/streaming/spawn_manager.py`,
`src/meridian/lib/harness/connections/resident_backend.py`

---


## Test Env Isolation: Private `_MERIDIAN_*` Vars Leak Across Session Boundaries

**The bug:** Tests run from inside a Meridian-managed session (e.g. a spawned
agent running `pytest`) inherited `_MERIDIAN_DEPTH` and `_MERIDIAN_HARNESS`
from the parent session. These private env vars silently changed runtime
behavior: `_MERIDIAN_DEPTH` caused the reaper to skip reconciliation (depth
gating invariant), and `_MERIDIAN_HARNESS` affected harness identity detection.
Reaper-scope integration tests that worked in local CI failed when run from a
Meridian spawn.

**Why it burned two cycles:** The conftest fixture cleared `MERIDIAN_*` (public)
env vars but not `_MERIDIAN_*` (private internal) vars. The first investigation
cycle (#456) identified the reaper test failure but attributed it to a state
bug. The second cycle (during PR #460 review) reproduced the same false
positives and traced them to the inherited private env vars. Both cycles would
have been avoided by clearing the full variable family from the start.

**The fix (PR #459):** The global conftest fixture now clears all `_MERIDIAN_*`
private vars in addition to `MERIDIAN_*` public vars. This ensures tests run in
a clean env regardless of whether they are launched from inside a Meridian
session.

**The lesson:** Private/internal env vars are still env vars. A test fixture that
clears the public namespace but not the private namespace creates a boundary
mismatch: production code reads private vars that tests did not neutralize.
When adding new `_MERIDIAN_*` vars to the runtime, verify the test conftest
clears them. The conftest clear pattern should match the full `_MERIDIAN_*` +
`MERIDIAN_*` family, not just the public surface.

**Where this lives:** `tests/conftest.py` (global fixture),
`src/meridian/lib/launch/env.py` (where private vars are set)

---

## Cursor: Stale Mars Binary Leaves harness_model Unresolved

**The scenario:** When testing cursor spawns in a worktree, the worktree had a stale
`mars` binary that predated cursor probe support (mars PR #72). Spawns appeared to
succeed, but `routing.harness_model` was absent from the launch bundle — the old
binary had no effort resolution logic and did not set the field.

**The silent failure:** When `harness_model` is absent, meridian falls back to
passing the raw model string (e.g. `--model gpt-5.5`) to cursor with no effort
suffix. No warning is logged. Cursor picks its own default effort tier, which may
not match what was requested. The subprocess command looks plausible but the effort
is silently wrong.

**Detection:** Run `meridian spawn --dry-run` and inspect the `--model` argument in
the emitted command. Check that `routing.harness_model` is set (non-empty) in the
launch bundle. If it is absent, the mars binary predates probe support.

**Fix:** Bump the mars-agents version pin in `mars.toml` and run `meridian mars sync`
in the worktree. Confirm `routing.harness_model` is non-empty in a dry-run spawn
before trusting effort projection.

**The lesson:** When shipping effort resolution for a harness across two repos (mars +
meridian), always confirm the worktree binary is the probe-capable version before
testing. The failure — wrong or missing effort suffix — is invisible unless you
inspect the bundle or the actual subprocess command directly.

**Where this lives:** `src/build/policy/runnable.rs:resolve_cursor_effort_slug()` (mars-agents) — sets `routing.harness_model`.

---

## Optional Defaults on Widened Seams Mask Missed Callers

**The bug:** When #171 widened the `resolve_session_file()` seam to accept a
`config_root_hint` parameter, the hint was given a default of `= None`. The
implementation threaded the hint through the primary resolution paths but
missed several `session repair` callers. Because the parameter defaulted to
`None`, the missed callers compiled and ran without error — they just silently
used ambient-only resolution, which was the original broken behavior.

**Why it burned a review cycle:** The adversarial reviewer wrote a
registry-wide runtime probe and found no `TypeError`, which appeared to
confirm totality. The `= None` defaults meant that even callers that should
have passed metadata compiled cleanly. The reviewer then traced the repair
paths manually and found the gap.

**The fix:** Make the hint parameter required (no `= None` default) on
internal helpers that carry it. Every caller must explicitly choose between
tracked metadata and untracked `None`. The missed repair callers became
immediate type errors rather than silent degradation.

**The lesson:** When widening a seam with a parameter that has a "do nothing"
value (`None`, `False`, empty), making it optional with a default hides
callers that should have been forced to choose. Required parameters turn
forgotten callers into compilation failures. This applies to any seam
widening where the new parameter controls correctness, not just convenience.

---

## Codex Live-Fork Rollout Snapshots

**The failure (probe-proven):** Forking a live Codex session used to clone
partial JSONL records. `CodexHarness.fork_session` read the rollout without a
fixed boundary, copied an unterminated final line from the live writer, and
could register the malformed fork in Codex's `state_5.sqlite`. The corruption
surfaced only when the fork was resumed and Codex parsed the final record.

Codex is the only harness where Meridian materializes the fork transcript.
Claude, OpenCode, and Pi delegate fork to the harness binary (`--fork-session`,
`--fork`, `--session --fork`), so their concurrent-write semantics are
harness-owned and not subject to this bug.

**Why it's subtle:** The copy reads to EOF without bounding to complete
records. A writer mid-line produces a well-formed file prefix followed by a
truncated JSON object. The insert into Codex's SQLite succeeds because the
DB row does not validate rollout content. The corruption surfaces only when
the forked session is later resumed and the harness tries to parse the last
record.

**Implemented contract:** `materialize_fork_rollout(...)` opens the source,
uses `fstat` on that descriptor as a snapshot bound, and streams at most that
many bytes through an atomic target. It copies only complete
newline-terminated records, validates each record, rewrites the first
`session_meta` id, and drops an incomplete tail at the bound. Memory therefore
scales with one record rather than the whole rollout.

File publication and SQLite registration are compensated rather than treated
as an impossible cross-store transaction. If registration raises, Meridian
checks the database again. A committed row wins; a target known to be
unregistered is removed; failure to determine commit state preserves the file
for recovery instead of risking deletion of a committed fork.

**The lesson:** When one side of a file is append-only live and the other
side copies it, the copy boundary must be a complete-records-only snapshot,
not a raw byte copy to EOF. `fstat` on the open descriptor gives an
immune-to-rename bound; truncation to the last newline gives record
completeness. This is the same discipline as crash-only JSONL reads —
partial trailing lines are expected and dropped.

**Where this lives:** `src/meridian/lib/harness/codex_rollout.py` (owns
rollout discovery/parsing), `src/meridian/lib/harness/codex.py`
(`CodexHarness.fork_session`)

---

## Green Suites Did Not Find the Seam Defects

**What happened:** During the native-session-identity work, every lane passed a green
full suite (~1,950 tests, pyright 0). Each lane still had defects that only showed up
at a seam between components:

- A Codex fork lost its recorded store in the fork *request builder*. The adapter-level
  fork tests passed because they called the adapter directly.
- Claude preparation seeded a fork from a same-ID decoy in the ambient root. Reads were
  exact, but nobody had put a decoy next to the recorded file.
- Codex bound a session ID from assistant prose (`codex resume <uuid>` in model
  output). Fixtures never put identity-shaped text in content.
- The Pi session-boundary extension poisoned its record on a real session switch. A
  read-only audit of Pi's installed lifecycle API and a Node fixture replaying the
  documented event order both passed. Only a real zero-turn Pi process showed that
  Pi invalidates the hook context on replacement, and that a shutdown can arrive on
  the invalidated runner when stdin closes mid-replacement.

**What found them:** an independent per-lane review, then a *recheck* of the fix pass
on a frozen head. The recheck found a new P1 that the fix pass had not touched.
Reviewers reproduced each defect in an isolated installed wheel using POSIX `sh`
harness shims and decoy fixtures, not source imports. The Pi defect needed the
real binary.

**The lesson:** For identity and lifecycle code, a green suite shows that the paths
you wrote down work. It says nothing about composition paths you did not enumerate.
Budget one review plus one recheck per lane. Qualify through the installed artifact
with adversarial fixtures: decoys, prose, nested keys, a changed ambient root. A fake
of an external lifecycle must reproduce its *invalidation* semantics (what stops
working after a transition), not just its event order. Before trusting an extension
against a host, run one real-process probe of the host lifecycle.

**Where this lives:** `tests/integration/launch/test_owned_identity_shims.py`,
`test_recorded_source_store.py`, `test_fork_source_store.py`,
`test_claude_recorded_prelaunch.py`, `test_pi_run_boundary.py`;
`pi_runtime/extensions/session-boundary/lifecycle.fixture.mjs`. Provenance:
`work:native-harness-session-identity` (`spawn:p7056`, `spawn:p7062`, `spawn:p7065`).

---

## When a Fix Pass Is Not Converging, Remove Carriers

**What happened:** After the first Lane C fix pass, the recheck still blocked, with one
old P1 partly open and one new P1. Both came from the same thing: three request fields
described "the recorded source" (`source_native_store`, a Claude config-root field, a
Pi session-dir field). Each had its own producer and consumers. One consumer (the fork
builder) forgot one field. Another (Claude preparation) gave a field a different
meaning than its producer did.

**The tempting fix:** patch the fork builder, patch Claude preparation, and add tests
for both. That would have been a third round of the same shape.

**What worked:** treat both P1s as one consolidation. Delete the two extra fields, make
`source_native_store` the only carrier, and push each harness's interpretation of the
store into its adapter. Every source read became exact, and the two P1s closed
without adding any mechanism. The check that the design was still converging was
simple: each cycle *removed* carriers rather than adding them.

**The lesson:** When review rounds keep finding "this consumer dropped/misread that
field," the fields are the defect. Count the carriers of one concept, reduce them to
one, and move per-consumer interpretation to the seam that owns the consumer.

---

## Cross-References

- [principles/design-principles.md](../principles/design-principles.md) — harness-agnostic, separate policy from mechanism
- [principles/invariants.md](../principles/invariants.md) — strategy map invariant, I-9 (no surface-specific logic in adapters)
- [architecture/launch-system.md](../architecture/launch-system.md) — three driving adapters and the composition factory
- [concepts/harness-abstraction.md](../concepts/harness-abstraction.md) — the harness abstraction model
- [architecture/pi-lifecycle.md](../architecture/pi-lifecycle.md) — Pi quiescence model and extension architecture
- [architecture/cursor-harness.md](../architecture/cursor-harness.md) — cursor probe design, raw-slug pattern, effort projection
- [architecture/native-session-binding.md](../architecture/native-session-binding.md) — native identity seams, source key, run boundary
