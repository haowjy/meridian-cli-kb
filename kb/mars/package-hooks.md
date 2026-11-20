# Portable hooks in packages

Packages can ship hooks that Mars renders into each harness's native hook files. Schema, target behavior and the supported answers: mars-agents `docs/config/mcp-and-hooks.md`. Hook importers and WASM import exports aren't built.

## One model, Claude's contract

Hooks go through the same canonical-model and target-rendering pipeline as agents and skills. A guard-only format would cover one pre-tool veto and need another schema for every later hook type. The portable syntax and runtime input and output follow Claude's hook contract, because Claude and Codex already run Claude-shaped scripts directly; a neutral protocol would need a translator process whose runtime might not be installed. `agy`'s events, payloads and replies differ from Claude's, so its hooks do go through a translator: the hidden `mars hook run`, named by absolute path, because the `mars` that ran the sync is the one runtime certain to be there (a generated shell script can't parse JSON, and Node or Python may be missing). The cost is that `.agy/hooks.json` is machine-local. A target reports what it can't carry. Codex can't enforce `onFailure = "block"`, so a script that must fail closed handles its own failure.

`mars-schema` owns the portable rules and rendering so WASM consumers see the same behavior as the binary; the binary owns installation and locking. Claude agent and skill frontmatter hooks stay out of the model: their lifetime is scoped and other harnesses have no equivalent.

## Rendering never drops a guard

- **Render only handlers a target parses.** One invalid Codex handler can make Codex discard the whole hooks file, user guards included.
- **An invalid dependency hook keeps its last good install**, in the same event-array position: dropping it could silently remove a guard, and moving it can invalidate Codex trust. Project-authored hooks are strict, so typos show.
- **Native hooks replace generated entries per event.** On merge targets a native hook replaces generated entries only for the events it declares, and an empty event array turns that event off for that hook. Replacing every generated event would silently remove unrelated guards; merging two definitions of one event would give one lock key two writers. On file targets the two coexist. OpenCode plugin trees carry generated hooks too, or a guard present under `mars sync` would vanish for plugin users.

## Harness behavior to recheck after upgrades

- Codex keys hook trust by file, event and array position. Its parser rejects a whole answer on an unknown field and can let the action run. Quoted dotted `projects` paths in `-c` can split wrongly; the inline-table form works.
- OpenCode's `PermissionRequest` fires only for `ask`, and edit guards must inspect patch text; its command-failure and compaction events don't match Claude's exactly.
- `agy` 1.5.13 reads a `{}` reply to `PreToolUse` as blocking every tool, so the runner answers no opinion with empty output. It fires `SessionStart` on a conversation's first prompt, not when the TUI opens and not on resume. A plugin's MCP tools weren't usable on the first prompt in two probes and were from the second. Its `Stop` payload's `continuationActive` is always false. Details: mars-agents `docs/config/mcp-and-hooks.md`, "Antigravity".
- Claude before 2.1.295 ignores `onFailure`. Pi can emit session start twice when replacing a session.

Open in mars-agents: sync diff for edited generated files ([#301](https://github.com/haowjy/mars-agents/issues/301)), lock source paths ([#302](https://github.com/haowjy/mars-agents/issues/302)), removing the deprecated alias ([#303](https://github.com/haowjy/mars-agents/issues/303)).
