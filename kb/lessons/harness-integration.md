# lessons/harness-integration — Lessons from Integrating Multiple AI Harnesses

These are the non-obvious discoveries from integrating Claude, Codex, and OpenCode into a single harness-agnostic coordination layer. Each lesson came from a real integration challenge.

## A Terminal Relay Is Not an Observation Channel

**The belief that outlived its reason:** "Claude needs the PTY so Meridian can
observe the session." Earlier designs said so: the session id was scraped from
the TTY stream. That stopped being true long ago. A fresh Claude session gets
its id through `--session-id <uuid>` before exec, idle reads the transcript,
and interactive output is never captured (only `--print` captures). Codex and
OpenCode were relayed too, though Meridian watches them through its own backend
connection.

**What it cost:** the nested PTY mangled resizes (#596). Its restore codes
overwrote harness exit text (the double `ESC[?1049l`, fixed in PR #607), and an
`ESC[J` workaround then erased that text. A component kept "for observation"
read nothing and caused bugs at the user-visible surface.

**The lesson:** when a mechanism's justification is "we need it to observe X",
check that X is still read through it. Here the honest answers are: Codex and
OpenCode need no relay (they now run native), and Claude/Pi keep it only
because the owner saw no resize problems with them, not because anything
depends on it. Keep terminal transport separate from
[identity binding](../architecture/native-session-binding.md). Current model:
[primary terminal ownership](../architecture/primary-terminal-ownership.md).

---

## Why Managed Primary Attach for Codex

**Codex's architecture:** Codex is not a traditional CLI tool that you invoke and wait for. It's an app-server model: a persistent HTTP/WebSocket server that manages agent state, with a TUI that connects to it as a client.

**The constraint:** Meridian needs to track the session, inject context, and observe outputs. It can't do this by wrapping a black-box process.

**The solution:** Managed primary attach. Meridian:
1. Starts the Codex app-server
2. Gets an observer connection (WebSocket)
3. Attaches the TUI to the server
4. Feeds each event from the observer connection through inline hooks (the attempt fold, harness event sinks), then to subscribers. Nothing is written to a runner stream ([native-only history](../decisions/native-only-history.md)).

The user sees the normal TUI. Meridian sees all events.

**The lesson:** Different harnesses have different architectural models. Claude is "invoke one black-box process." Codex is "connect to a running server." The adapter layer must accommodate both, and the connection machinery (`harness/connections/`) enables this without exposing the difference to the rest of the system.

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

**The second half (2026-10-15):** main-thread installation is not enough. A
Python signal handler also *runs* on the main thread, between bytecodes of
whatever the main thread was doing. The native-terminal launcher's
`_TuiExitOrdering` recorder took a plain `threading.Lock` both from main-thread
code and from a `SignalCoordinator` receiver. A SIGHUP arriving while the main
thread held the lock would make the handler wait on a lock its own thread
holds, so the launcher would hang on a pane close. Review caught it and the
lock became an `RLock`. Rule: anything a signal receiver touches must be
lock-free or reentrant, and every handler goes through `SignalCoordinator`
rather than raw `signal.signal`. A raw `SIG_IGN` for SIGINT once bypassed it
and could lose the handler restore.

**Where this manifests:** `launch/signals.py` (signal coordination), `launch/process/pty_launcher.py` (PTY launch with main-thread signal setup), `launch/process/primary_attach.py` (signal receivers, `_TuiExitOrdering`)

---


## Why Extension Injection for Pi (Managed-Bash and Spawn Watch)

**The problem:** Pi's JSONL RPC event stream does not provide durable background-task ownership, canonical child membership, or proof that a queued follow-up was admitted and observed. Meridian needs those facts before declaring a spawned session done.

**The solution:** Meridian injects TypeScript extensions into Pi via `-e <path>` flags. These extensions run inside Pi's process, access Pi's bash execution context, and write durable coordination state. Python watches that disk state to decide quiescence.

Two managed extensions supply the completion evidence:
- **`managed-bash`** — overrides Pi's bash builtin. Registers `bash` tool (with `background?: boolean` + `timeout_min?: 1-59` parameters) and `bash_manage` ops tool. Returns immediately with `{bash_id, status: "backgrounded"}` for tracked bg transitions; blocks for synchronous calls.
- **`meridian-spawn-watch`** — observes canonical direct child rows, dispatches implicit-wait follow-ups (`sendMessage({triggerTurn: true})` at idle, plus native-steer delivery at tool-bearing turn boundaries since v0.9.4/PR #560; see [Pi result-delivery decision](../decisions/pi-runtime.md)), records exact native admission and delivery faults, and provides the `/spawn` family of slash commands (renamed from `/mspawn` — no compatibility alias).

**Why disk-state observation:** `managed-bash` writes `pi-bash/<spawn-id>/bash-records.json`; the watcher writes exact `delivery-receipts.json`; Python writes matching `delivery-observations.json`; and Meridian's spawn store writes `runtime_root/spawns/<child>/state.json`. Python-side `PiDiskWatcher` wakes completion policy when private evidence changes. `parent_id` defines child membership; `originating_bash_id` transfers a launcher obligation but cannot establish membership. `last-notification.json` and origin sidecars do not supply authority.

**The lesson:** When a harness's native protocol doesn't expose enough structure for task ownership and completion evidence, extension injection lets Meridian add a code path inside the harness process rather than working around it. The tradeoff is coupling to Pi's extension API: if Pi changes its extension loading mechanism, the extensions may need updates. The compatibility probe (`pi --help` must advertise `--no-extensions` and `-e`) gates the harness before any spawn. Exact native admission and Python event observation remain separate because they cannot commit atomically across a process crash.

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
closed the parent early; if a child was still launching a managed backend,
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

**The same hole, one namespace over (2026-09-25).** Tests that swap `HOME` still
inherited `CODEX_HOME` from the Codex-run spawn executing them. They opened the
user's real `~/.codex` SQLite read-write; they failed before writing, only by luck.
Harness store variables are as ambient as `_MERIDIAN_*`. The autouse fixture now
also clears `_NATIVE_STORE_ENV`:
- `CODEX_HOME`, `CLAUDE_CONFIG_DIR`;
- `OPENCODE_DB`, `OPENCODE_CONFIG`, `OPENCODE_CONFIG_DIR`;
- `PI_CODING_AGENT_DIR`, `PI_CODING_AGENT_SESSION_DIR`;
- `XDG_CONFIG_HOME`, `XDG_DATA_HOME`, `XDG_STATE_HOME`.

The suite was then re-run with these variables pointed at an unreadable trap
directory to prove nothing still reaches them. When an adapter starts reading a new
store variable, add it to that set.
Provenance: `work:native-harness-session-identity` (`spawn:p7100`, commit
`7c49b1d5`).

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

## A Function-Local Import Hides a Layering Cycle

**The bug:** The Codex idle parser in `lib/harness/codex_idle.py` had to know
whether a `notify` event's thread was a pinned primary, which lives in idle state.
Its default `session_reader` built an `IdleService` through a function-local
`from meridian.lib.idle.service import IdleService`. `lib/idle` already imports
`lib/harness/idle_types`, so this made a `harness ↔ idle` cycle. Because the import
ran only at call time, nothing failed at import, and every test passed.

**The fix:** `meridian idle event` supplies the native hook's `session_reader`.
The parser no longer constructs a policy service. An AST test checks every
module under `lib/harness`, including function bodies.

**The lesson:** a layering rule that only import-time checks enforce is not
enforced. Function-local imports and defaulted callbacks are the two ways around
it. Inject the dependency without a default, as in the section above, and enforce
the direction with an AST test that includes function bodies. See the
[idle dependency boundary](../decisions/idle-notify-channels.md#d-idle-leaf-packages--libnotify-and-libidle-are-leaf-services).
Provenance: `work:idle-cache-notify`, `reviews/G2-review.md` K3.

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

## Bundled Mods Outlive the CLI They Call

**The mechanism:** Meridian ships in-harness code inside the meridian-cli
package: the Claude mods under `claude_runtime/` (passed with `--plugin-dir`,
resolved from the installed package by `claude_runtime_resolver.py` at launch)
and the Pi extensions (passed with `-e`). The harness loads that code once, at
session start. Upgrading meridian-cli replaces the CLI on PATH at once, but every
interactive session that is already running keeps the mod it loaded. Until the
session restarts, that old mod calls the new CLI.

**What happened (v0.9.8, 2026-10-14):** PR #568 removed `--cwd` from
`meridian idle arm`
([D-idle-ttl-detection](../decisions/idle-ttl-detection.md#d-idle-ttl-detection--detection-reads-launch-records-never-a-cwd)).
In a session started on 0.9.7, the mod still sent `--cwd` and got
`{"error":"Unknown option: --cwd."}` with exit 1. The Claude mod's `note()`
writes failures only to the mod's own store log (`arm failed: …`), so nothing
showed in the TUI. Idle push, warn and compact all stopped for that session.
The change was accepted under "no backwards compatibility", and restarting
running primaries went to the user as an upgrade note.

**The lesson:** a change to a CLI flag or reply shape that a bundled mod relies
on is a contract break for every running session, not only for new launches.
The idle mods report failures quietly by design, so the break shows up as idle
features that stop firing, with no error message. When such a change ships,
its upgrade note needs to tell users to restart running interactive sessions.

**Where this lives:** `src/meridian/claude_runtime/meridian-idle/hooks/register.ts`
(`note()`, the `arm` call), `src/meridian/pi_runtime/extensions/meridian-idle/`,
`src/meridian/lib/harness/claude_runtime_resolver.py`. Provenance:
`work:idle-ttl-detection` `decisions.md` "2026-10-14 post-merge" and "Outcome".

---

## A Dialect Switch Must Move Every Consumer of the Stream

**The mechanism:** OpenCode 2.x renamed its turn-end event. Meridian's V2
transport (v0.6.0, 2026-09-18) handled the rename where the spawn path reads
it: `OPENCODE_SEMANTICS` gained `session.execution.{succeeded,failed,interrupted}`
entries, and `connections/AGENTS.md` and the harness `.context/CONTEXT.md`
recorded that V2 ends turns that way. Three weeks later the idle sensor (PR #550,
v0.9.3) was built as a second consumer of the same live stream. It matched event
names itself instead of reading the semantics table. It was designed and probed
against OpenCode 1.18.32, a V1 backend, and its unit fixtures were V1 SSE
captures only. It waited for `session.idle` and compacted through V1's
`/session/{id}/summarize`. Meanwhile the general harness docs still said
"OpenCode uses `session.idle`" (`lib/harness/AGENTS.md` and two lines of the
harness `.context/CONTEXT.md`; corrected on the PR #573 branch).

**What happened:** on OpenCode 2.0.28 the sensor waited for an event that
never came. Nothing raised. No arm call meant no idle state file, and the
sidecar's per-event fault containment had nothing to contain, so there was no
`debug.jsonl` either. All 10 interactive OpenCode primaries on Oct 12–13 (one
three hours long) got no push, no warning and no compaction, through every
release from 0.9.3 to 0.9.8. The #550 review had listed "if `session.status` /
`session.idle` change, there are no arms" as fails-closed. For idle, fails-closed
is fails-silent. It surfaced only when the user re-opened idle work for the
other harnesses and the before-state check found no OpenCode idle state at
all. Fix:
[D-opencode-idle-v2](../decisions/idle-opencode-v2.md#d-opencode-idle-v2--opencode-2x-gets-its-own-sensor-chosen-by-the-resolved-transport)
(PR #573).

**The lesson:**

- When a harness changes dialect, list every consumer of its event stream, not
  just the transport. A consumer that matches event names itself does not
  inherit the transport's update.
- Probe a new adapter against the harness version users actually run, and keep
  fixtures for every dialect the dispatcher can select. A V1-only fixture set
  passes forever against a V2 world.
- A sensor whose failure mode is "never fires" needs a positive signal that it
  has fired, such as a real session's `meridian idle status` row. Absence of
  errors is not evidence.
- When the dialect is chosen at runtime, the consumer must follow the
  transport's resolved choice, not re-read the preference. The first V2 fix
  dispatched on the env var, which carries `auto`, and sent 1.x users the V2
  sensor (review-2 H1).

**Sibling, not the same:**
[Bundled mods outlive the CLI they call](#bundled-mods-outlive-the-cli-they-call)
is also silent idle loss, but there the two sides of Meridian's own contract
drift across an upgrade. Here Meridian never matched the harness version in
the first place.

**Where this lives:** `src/meridian/lib/harness/opencode_idle.py`
(`primary_idle_sensor`, both sensors),
`src/meridian/lib/harness/connections/opencode_connection.py`
(`resolved_version`), `src/meridian/lib/harness/opencode.py`
(`OPENCODE_SEMANTICS`). Provenance: `work:idle-ttl-detection` `decisions.md`
"2026-10-14 phase 2", `evidence/probe-opencode-v2-events.md`,
`reviews/review-2.md`; `work:idle-cache-notify` `probes/P0c.md` (1.18.32) and
`reviews/G2-review.md`.

---

## Cross-References

- [principles/design-principles.md](../principles/design-principles.md) — harness-agnostic, separate policy from mechanism
- [principles/invariants.md](../principles/invariants.md) — strategy map invariant, I-9 (no surface-specific logic in adapters)
- [architecture/launch-system.md](../architecture/launch-system.md) — three driving adapters and the composition factory
- [concepts/harness-abstraction.md](../concepts/harness-abstraction.md) — the harness abstraction model
- [architecture/pi-lifecycle.md](../architecture/pi-lifecycle.md) — Pi quiescence model and extension architecture
- [architecture/cursor-harness.md](../architecture/cursor-harness.md) — cursor probe design, raw-slug pattern, effort projection
- [architecture/native-session-binding.md](../architecture/native-session-binding.md) — native identity seams, source key, run boundary
- [decisions/native-session-identity.md](../decisions/native-session-identity.md)
