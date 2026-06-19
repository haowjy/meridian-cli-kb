# Architecture: Claude Session Isolation

Meridian isolates each Claude primary and child spawn in its own config-dir
overlay so that concurrent sessions in the same project cannot bleed into each
other. This page explains why the isolation is necessary, how the overlay
mechanism works, and how transcripts and selected auth/config state survive the cleanup lifecycle.

Related pages:
- [codebase/harness-adapters.md](../codebase/harness-adapters.md) — Claude adapter PTY capture, system-prompt channel
- [operations/health-checks.md](../operations/health-checks.md) — doctor overlay pruning
- [concepts/spawn-lifecycle.md](../concepts/spawn-lifecycle.md) — spawn status transitions

---

## The Upstream Limitation

Claude Code stores session transcripts at:

```
<config-root>/projects/<slug>/<session-id>.jsonl
```

where `<config-root>` defaults to `~/.claude/` and `<slug>` is a sanitized
encoding of the working directory path. All Claude processes that share the
same `CLAUDE_CONFIG_DIR` (the default) share the same session store.

This creates **session bleed** when two or more Meridian sessions run
concurrently in the same project directory:

1. **Heuristic fallback detection** — if Claude loses its session ID, it scans
   `projects/<slug>/` for the most recent `.jsonl` file. With two concurrent
   sessions, it may pick up the other session's transcript.
2. **Prefix-matched slug resolution** — Claude may match a `my-project-v2/`
   directory when looking for `my-project/`, causing cross-project association.
3. **Inherited `CLAUDE_CONFIG_DIR`** — a nested `meridian spawn` child inherits
   the parent process's `CLAUDE_CONFIG_DIR`, putting child and parent transcripts
   in the same namespace.
4. **Session ID discovery gap** — child spawns that lack a pre-seeded session ID
   must scan for their own session file, which can find the wrong one.

Codex and OpenCode are not affected: they use different session-store models
and do not share `~/.claude/projects/`.

---

## The Overlay Approach

For each Claude session — primary or child spawn — Meridian creates an
**isolated config-dir overlay** before launching the harness:

```
~/.meridian/projects/<project-uuid>/claude-config/<spawn-id>/
  settings.json           → linked or copied from ~/.claude/settings.json
  settings.local.json     → linked or copied from ~/.claude/settings.local.json
  permissions.json        → linked or copied from ~/.claude/permissions.json
  mcp/                    → linked directory or copied directory from ~/.claude/mcp/
  projects/               (empty, fresh for this session)
```

The overlay contains **links and/or copied files from the user's real
settings** (symlinks are common on POSIX; Windows may use copied files and
directory-level link primitives) and a **fresh empty `projects/` directory** for
this session's transcripts. `CLAUDE_CONFIG_DIR` is set to the overlay path in
the harness's env, so Claude writes transcripts there and never sees other
sessions' files.

**Shared helper:** `prepare_isolated_claude_config()` in
`lib/harness/claude_preflight.py` creates the overlay. Both primaries and child
spawns call the same function — there is no separate mechanism (EARS-ISO-109).
If overlay creation fails (e.g., cannot create temp directory), the launch falls
back to the shared config dir with a logged warning rather than failing
(EARS-ISO-102).

---

## Primary vs Child Behavior

Both get isolated overlays, but the lifecycle owner differs:

| | Primary | Child spawn |
|---|---|---|
| Overlay created by | `runner.py` (primary launch path) | `launch_prepared_spawn()` in `execute.py` |
| Overlay path recorded in | spawn record + session record | spawn record + session record |
| Transcript seeding owner | `runner.py` before exec | `launch_prepared_spawn()` at overlay creation |
| Cleanup on exit | `runner.py` `finally` block | `launch_prepared_spawn()` `finally` block |
| Inherited from parent | Never — child env scrubs parent's value (EARS-ISO-104) | N/A |

**Child spawn isolation detail:** A `meridian spawn` process running inside an
isolated primary inherits `CLAUDE_CONFIG_DIR` from the parent process env.
Meridian scrubs this value and creates a fresh overlay for the child — children
get their own namespace, not the parent's (EARS-ISO-104, EARS-ISO-108).

**Transcript seeding for continue/fork:** When a child spawn continues or forks
from an isolated source, `ensure_claude_session_accessible()` is called inside
`launch_prepared_spawn()` at overlay creation time, passing both
`source_config_root` (where the source transcript lives) and
`target_config_root` (the child's new overlay). This happens before execution
starts so the harness finds the transcript on launch.

---

## Transcript Materialization

Overlay cleanup is the critical lifecycle step. Simply deleting the overlay
after a session exits would destroy the transcript — `--continue` would fail
because Claude cannot find the session file.

Before `shutil.rmtree(overlay_root)`, Meridian calls:

```python
materialize_overlay_transcripts(overlay_root, canonical_root=None)
```

This copies every `.jsonl` file from `<overlay>/projects/<slug>/` into
`<canonical>/projects/<slug>/`, where `<canonical>` is the real `~/.claude/`
(or whatever `CLAUDE_CONFIG_DIR` resolves to for a non-overlay launch). After
materialization, the transcript is at the same location it would occupy in a
non-isolated launch — transparent to users and to Claude's session-discovery
logic.

**Overwrite policy:** The overlay copy wins when its `mtime` is newer than the
canonical copy. If `mtime` values are equal (coarse timestamp resolution or
NFS), the larger file wins. This handles multi-resume chains correctly: each
continued session creates a new overlay and produces a longer transcript; on
cleanup the newer overlay copy overwrites the older canonical copy.

**Error handling:** Materialization failures are logged at `WARNING` level and
do not propagate — transcript loss is preferable to a broken exit path
(EARS-ISO-115).

**Metadata repair on successful cleanup:** After successful materialization and
overlay cleanup/prune, Meridian updates spawn/session Claude-config metadata to
the durable canonical/materialization root rather than leaving it pointed at
the deleted overlay path. This keeps later continue/fork resolution on the
direct path without needing overlay-path recovery.

---

## Auth and Config State Persistence

Overlay isolation must not force repeated Claude authentication. Claude mutates
selected auth/config files during runtime, so cleanup preserves the known mutable
auth state from the overlay back into the durable materialization root before
deleting the overlay.

Known auth/config files materialized from overlays:

- `.claude.json`
- `.credentials.json`

`materialize_overlay_auth_state()` copies these files from the overlay root to
the durable Claude config root chosen for the overlay lifecycle. The same
materialization root is recorded in overlay metadata (`.meridian-overlay.json`)
so cleanup can still find the intended durable root even if the ambient
environment changed before cleanup runs.

**Concurrency policy:** Overlay isolation remains the primary safety boundary.
Each Claude process mutates its own copied auth files inside its overlay. During
cleanup, each known auth file is copied to a temporary file, guarded by the same
per-relative-path lock mechanism used for transcript materialization, then
atomically replaces the durable file only when the overlay copy is newer (or the
same `mtime` with larger size). This preserves the freshest auth/config state
without sharing a live mutable file between concurrent Claude sessions.

**Failure policy:** Auth materialization is best-effort. Failures are logged as
warnings and do not block transcript materialization or overlay removal. A
failure can cause future Claude launches to prompt for auth again, but should
not strand a spawn in cleanup.

---

## The `--continue` Flow After Isolation

Without materialization, `meridian --continue <session-id>` fails with
contradictory output because the session file exists in the canonical
`~/.claude/projects/` tree but Claude cannot find it (the overlay path
recorded in metadata was deleted).

With the full fix:

```
ORIGINAL SESSION:
  1. Overlay created at runtime_root/claude-config/<spawn-id>/
  2. CLAUDE_CONFIG_DIR → overlay
  3. Claude writes transcript to overlay/projects/<slug>/<id>.jsonl
  4. Claude exits
  5. materialize_overlay_transcripts() → copies to ~/.claude/projects/<slug>/<id>.jsonl
  6. shutil.rmtree(overlay)

CONTINUE SESSION:
  1. resolve_session_reference() finds source_control_root + source_claude_config_dir
  2. New overlay created; CLAUDE_CONFIG_DIR → new overlay
  3. ensure_claude_session_accessible() fires:
       - source_config_root = old overlay path (deleted) → falls back to canonical
       - canonical has the transcript (from step 5)
       - copies transcript into new overlay/projects/<slug>/
  4. Claude receives --resume <id>, finds transcript in new overlay
  5. Session continues
```

**Canonical fallback:** `ensure_claude_session_accessible()` first tries the
explicit `source_config_root`. For repaired metadata this is usually already
the canonical root, but fallback to `_claude_config_root()` remains important
for older records and partial-failure paths where metadata still points at a
deleted overlay. Both paths use `_claude_config_root()` rather than hardcoded
`~/.claude/` to respect a custom `CLAUDE_CONFIG_DIR`.

**Same-CWD fast-path suppression:** The same-CWD fast path (skip seeding when
source and target CWDs match) is suppressed when source and target config roots
differ — isolated sessions from the same directory still need transcript seeding
into the new overlay.

---

## Orphaned Overlays and Doctor Pruning

If Meridian crashes before the cleanup `finally` block runs, the overlay
directory is orphaned at `runtime_root/claude-config/<spawn-id>/`. The
transcript inside may never have been materialized.

`meridian doctor --prune` detects and removes stale overlays via
`scan_stale_claude_overlays()`. Before deleting each orphaned overlay, doctor
attempts `materialize_overlay_transcripts()` to rescue any transcripts the
crash-orphaned overlay contains. If materialization fails, doctor logs a
warning and continues prune handling (including overlay deletion) rather than
failing the overall prune pass.

Only overlays for inactive (non-running) spawns in the current project are
deleted. Retention semantics match existing spawn artifact pruning (respects
`state.retention_days`). See
[operations/health-checks.md](../operations/health-checks.md#claude-overlay-pruning)
for the full pruning table.

---

## Design Constraints

**Single helper, no dual mechanism.** The requirement that primaries and
children use `prepare_isolated_claude_config()` without a second alternative is
load-bearing. A second overlay mechanism would create divergent cleanup
semantics, different metadata schemas, and duplicate seeding logic. Any future
change to overlay layout or cleanup must go through the shared helper.

**Materialization to canonical, not a Meridian-owned archive.** Alternatives
considered:
- *Archive to Meridian dir*: Would require a new transcript index and custom
  resolution path. The canonical location is where Claude already looks —
  zero additional resolution logic.
- *Don't delete overlays; let doctor GC them*: Overlays accumulate quickly
  (5–10 per orchestrator run) and contain linked/copied config plus credential
  copies, not just transcripts. Indefinite retention creates a confusing state
  model.
- *Copy to canonical (chosen)*: After materialization, an isolated launch is
  indistinguishable from a non-isolated one from the user's perspective. The
  session file is exactly where Claude expects it.

**Preserve known auth files, not the whole overlay.** Cleanup copies only the
known mutable Claude auth/config files (`.claude.json`, `.credentials.json`) back
to the durable root. Copying the whole overlay would reintroduce session bleed by
persisting overlay-specific `projects/` state and linked/generated runtime
artifacts. Sharing auth files live between overlays would weaken isolation and
create concurrent mutation races.

**Canonical `projects/` mixes Meridian and user sessions.** This is
intentional — the canonical location is where Claude's session-discovery logic
looks. It means `meridian doctor` cannot safely prune files under
`~/.claude/projects/` (it is not a Meridian-owned directory). Stale transcripts
in the canonical location must be cleaned up through Claude's own tooling.

**Metadata converges to canonical root.** Successful cleanup/prune writes the
durable canonical/materialization root into spawn/session
`source_claude_config_dir` metadata. Canonical fallback remains in place for
pre-repair records and partial-failure paths where convergence did not happen.

---

## TUI Trampoline Session-ID Reconciliation

When Claude is launched via its new TUI, entering `/tui fullscreen` creates a
transient trampoline session with its own session ID. The actual conversation
continues under a new, different session ID — the durable transcript.

If Meridian records the trampoline session ID during launch (e.g., via PTY
capture or `--session-id` in args), later operations (`meridian session log`,
prior-run context injection) will fail: the trampoline ID has no transcript
file under `~/.claude/projects/<slug>/`.

**Reconciliation at finalization.** `ClaudeAdapter.observe_session_id()`
overrides the base implementation to detect and repair this case:

1. If the recorded session ID already has a transcript file → preserve it
   (most launches are unaffected).
2. If no transcript exists → scan `~/.claude/history.jsonl` for the recorded
   ID with `display: "/tui fullscreen"`.
3. Find the next same-project prompt entry with a different session ID.
4. Verify the successor has a transcript whose first user message matches the
   history display.
5. If all checks pass → update session and spawn records to the real ID.
6. If any check fails → fall back to the recorded ID (no guessing).

This runs during primary finalization in `runner.py`, after the harness exits.
It is Claude-specific — Codex, OpenCode, and Pi do not have a `/tui fullscreen`
trampoline pattern.

**Constraints that shaped the design:**

- **File-based only.** A `claude --resume` probe was tested and rejected: it
  performs model work that hits budget limits and is not deterministic.
  Reconciliation stays entirely within file reads.
- **Adapter override, not generic finalization.** The reconciliation is an
  `observe_session_id()` override in `ClaudeAdapter`, not a shared finalization
  step. This keeps trampoline-specific logic in the Claude module where
  transcript-path and `history.jsonl` helpers already live.
- **Conservative fallback.** When evidence is insufficient (no trampoline
  marker in history, successor transcript doesn't match, multiple candidates),
  the recorded ID is preserved. The system behaves exactly as before — fails or
  succeeds based on the recorded ID — rather than guessing.

**Related code:**
- `src/meridian/lib/harness/claude.py` — `reconcile_tui_trampoline_session_id()`,
  `_find_tui_trampoline_successor_session_id()`, `ClaudeAdapter.observe_session_id()`
- `src/meridian/lib/launch/process/runner.py` — calls `observe_session_id()` during
  primary finalization
- `tests/integration/harness/test_adapter_ownership.py` — reconciliation unit tests
- `tests/integration/launch/test_launch_process_claude_session.py` — end-to-end
  trampoline reconciliation test
