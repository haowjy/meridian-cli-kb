# The Rust rewrite

meridian-cli is to be rewritten in Rust; the project invariants in the root `AGENTS.md` hold through it. It has no settled design yet. Its first piece has shipped: the `meridian-sync` daemon ([context sync](../context-sync.md#rust-inside-meridian-cli)), which then folds into the one `meridian` binary. Two problems motivate the rewrite:

- **Memory.** The Python import floor is about 90 MB per process, so concurrent Meridian processes multiply the baseline. Algorithmic heap growth is a separate problem that a direct port of the same algorithm would keep.
- **Upgrades crash running processes.** `uv tool install --force` can overwrite modules inside a live process's virtualenv, and a later lazy import then mixes versions ([#628](https://github.com/haowjy/meridian-cli/issues/628)). Until the rewrite, avoid upgrading while long-lived Meridian processes run.

A static binary alone doesn't solve upgrades. Points the design will need to weigh: replacing the binary by rename protects running processes from partial replacement; not re-executing through a mutable path stops a process switching versions mid-run; embedded or versioned assets prevent mixed versions; and versioned shared state lets old and new processes coexist ([compatibility](../state/compatibility.md)). Process-cleanup untangling is deferred to it too ([#940](https://github.com/haowjy/meridian-cli/issues/940)). Chat and spawn IDs are to become machine-wide there; today they count per project, so a bare ID crossing projects can name the wrong session ([session references](../launch/session-references.md#ids-count-per-project)). Model fallback chains are deferred to it too. Mars resolves an auto-resolve alias to the newest matching catalog model by release date without checking availability, and the launch fallback chain only switches harnesses for that same model. When the chosen provider is unavailable there is no path to another provider serving the same model — a GLM alias resolves to z-ai on OpenRouter while the runnable copy is on opencode-go, and today that needs a locally pinned alias. Fallback that picks an available provider across models is rewrite work ([model resolution](../mars/model-resolution.md)).

## Meridian as an MCP server

Jimmy, 2026-11-20: the rewrite may make Meridian an MCP server. This is an idea for the design, not a decision.

**The shape: one tool over the CLI.** Expose a single tool, `meridian`, whose input is an argument list and whose description says to start with `["-h"]`. The agent discovers commands through `-h` and `--json` output, as it does with the CLI. The CLI stays the only definition, so there is no second surface to keep in step, and one tool definition costs little context. Some harnesses give every agent every MCP server; Antigravity (`agy`) does. The server runs the command from the argument list, never through a shell.

**What it fixes.**
- Identity. Today the caller is known from `MERIDIAN_*` environment variables, which break wherever a shell doesn't inherit them: a harness Meridian didn't launch, or a sandboxed shell. `agy` tags each MCP call's `_meta` with `antigravity.google/conversation_id`, `root_conversation_id` and, for its own subagents, `parent_conversation_id`. The server sets the session's variables per call. Whether Claude, Codex and OpenCode put a session ID in MCP `_meta` is unchecked.
- Sandboxes and approval. In `agy`, a shell command can't write `~/.meridian`, and a child started from that shell can't reach its API. The MCP server runs outside the terminal sandbox, and the harness's own per-tool approval prompt gates it.
- A per-session process. Meridian already needs one to watch a session and deliver messages. In Python it costs about 90 MB; in Rust it would be small, so one MCP server per harness process, also serving the control socket, is affordable.

**What it doesn't fix.**
- MCP can't push text to the model, so each harness still needs a delivery route: hooks, Claude's inbox, app-server connections.
- One tool means one approval covers every command, spawns included.
- Long waits must fit each harness's tool-call timeout, so wait in bounded calls.
- The CLI remains the contract for people, scripts and `meridian-comms`.

**What to carry into the design.**
- One registry of operations, with typed inputs and outputs, from which the CLI and JSON surfaces (and, if ever needed, MCP) are generated.
- The caller as an explicit input to every operation, resolved by each harness adapter from what that harness provides: MCP call data, environment variables or a hook payload. Not read from the environment deep in the code.
- One light process per session serving MCP and the control socket.

First instance: the `agy` integration's plugin MCP server, from the `antigravity-harness` work item. Its evidence is `probe-plugin-mcp.md`, on `agy` 1.5.13.
