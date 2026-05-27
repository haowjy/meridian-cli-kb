# lessons/harness-integration — Lessons from Integrating Multiple AI Harnesses

These are the non-obvious discoveries from integrating Claude, Codex, and OpenCode into a single harness-agnostic coordination layer. Each lesson came from a real integration challenge.

## Why PTY Capture for Claude Primary Launch

**The constraint:** Claude's interactive TUI emits its session ID to the terminal on startup. There's no flag to get the session ID programmatically, and it doesn't emit structured JSON on stdout during interactive mode.

**The naive approach:** Pipe stdout and parse for the session ID. This fails because the TUI is designed to talk to a terminal. Interactive behavior, colors, and in-place rendering require a TTY. Piping stdout gets garbled or suppressed output.

**The solution:** Launch Claude under a PTY (pseudo-terminal). The PTY convinces Claude it's talking to a real terminal. Meridian reads the PTY output, extracts the session ID from the startup banner, and then hands the PTY to the user's terminal so the interactive session proceeds normally.

**The lesson:** When a tool is designed for humans, observation requires meeting it on its terms. PTY capture is the minimum machinery needed to observe session ID without reimplementing Claude's TUI. It's explicitly limited to that extraction use case — not used as a general control mechanism.

**Where this lives:** `launch/process/pty_launcher.py`, `harness/adapters/claude.py:detect_primary_session_id()`

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

**The solution:** Meridian injects TypeScript extensions into Pi via `-e <path>` flags. These extensions run inside Pi's process, access Pi's internal event bus and bash execution context, and coordinate completion by watching spawn and bash records on disk from Python.

Two managed extensions ship as package data:
- **`managed-bash`** — overrides Pi's bash builtin. Registers `bash` tool (with `background?: boolean` + `timeout_min?: 1-59` parameters) and `bash_manage` ops tool. Returns immediately with `{bash_id, status: "backgrounded"}` for tracked bg transitions; blocks for synchronous calls.
- **`meridian-spawn-watch`** — watches spawn records and bash records on disk via `watchfiles`. Emits implicit-wait completion notifications (`sendMessage({triggerTurn: true})`) when tracked work terminates.

**Why disk-state observation instead of sidecar:** The redesign replaces the sidecar transport with direct disk observation. `meridian-spawn-watch` watches `~/.meridian/projects/<proj>/spawns/` and `pi-bash/<spawn-id>/bash-records.json` via `watchfiles`. Disk state is the canonical signal; env-var correlation (`MERIDIAN_PI_BASH_ID` → `originating_bash_id`) bridges bash and spawn observations without command-string parsing.

**The lesson:** When a harness's native protocol doesn't expose enough structure for quiescence detection, extension injection lets Meridian add a code path inside the harness process rather than working around it. The tradeoff is coupling to Pi's extension API: if Pi changes its extension loading mechanism, the extensions may need updates. The compatibility probe (`pi --help` must advertise `--no-extensions` and `-e`) gates the harness before any spawn.

**Where this lives:** `src/meridian/pi_runtime/extensions/` (managed-bash + meridian-spawn-watch), `harness/connections/pi_rpc.py`, `harness/pi.py:env_overrides()`

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

## Cross-References

- [principles/design-principles.md](../principles/design-principles.md) — harness-agnostic, separate policy from mechanism
- [principles/invariants.md](../principles/invariants.md) — strategy map invariant, I-9 (no surface-specific logic in adapters)
- [architecture/launch-system.md](../architecture/launch-system.md) — four driving adapters and the composition factory
- [concepts/harness-abstraction.md](../concepts/harness-abstraction.md) — the harness abstraction model
- [architecture/pi-lifecycle.md](../architecture/pi-lifecycle.md) — Pi quiescence model and extension architecture
- [architecture/cursor-harness.md](../architecture/cursor-harness.md) — cursor probe design, raw-slug pattern, effort projection
