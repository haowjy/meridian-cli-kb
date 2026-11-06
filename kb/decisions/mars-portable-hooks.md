# Mars Portable Hooks

Hook importers, WASM import exports and Cursor rendering are not built. The schema, target behavior and supported answers are in mars-agents `docs/config/mcp-and-hooks.md`.

## Hook model and syntax

Hooks follow the same canonical-model and target-rendering pipeline as agents and skills. A guard-only format would solve one pre-tool veto but require another schema for later hook types. The portable syntax and runtime input/output use Claude's hook contract: Claude, Codex and Cursor already run Claude-shaped scripts directly, and a neutral protocol would require a translator process whose runtime might not be installed. A target reports what it cannot carry. Codex cannot enforce `onFailure = "block"`; scripts requiring fail-closed behavior must handle failure themselves.

`mars-schema` owns portable rules and rendering so WASM consumers see the same behavior as the binary; the binary owns installation and locking. Claude agent and skill frontmatter hooks remain out of the model because their lifetime is scoped and other harnesses lack an equivalent.

## Render and installation safety

Render only handlers a target parses: a single invalid Codex handler can cause it to discard the entire hooks file, including user guards. An invalid dependency hook keeps its last good install, including its event-array position; dropping it could silently remove a guard, and moving it can invalidate Codex trust. Project-authored hooks are strict so typos are visible.

On merge targets a native hook replaces generated entries for only the events it declares; an empty event array turns that event off for that hook. Replacing all generated events would silently remove unrelated guards, while merging two definitions for the same event gives one lock key two writers. On file targets generated and native hooks coexist. OpenCode plugin trees must carry generated hooks too, or a guard present under `mars sync` disappears for plugin users.

## Harness hazards

Observed behavior is version-specific; recheck after harness changes:

- Codex trust is keyed by file, event and array position. Its parser rejects a whole answer on unknown fields and can let an action run. Quoted dotted `projects` paths in `-c` can split incorrectly; the inline table form was verified.
- OpenCode `PermissionRequest` fires only for `ask`, and edit guards must inspect patch text. Its command-failure and compaction events do not exactly match Claude's.
- Claude versions before 2.1.295 ignore `onFailure`. Pi can emit session start twice when replacing a session.

Follow-ups: [#301](https://github.com/haowjy/mars-agents/issues/301) (sync diff for edited generated files), [#302](https://github.com/haowjy/mars-agents/issues/302) (lock source paths), and [#303](https://github.com/haowjy/mars-agents/issues/303) (remove the deprecated alias).
