# Meridian KB

Why meridian-cli, Mars and the surrounding packages are the way they are. Directory contracts live beside the code in the source checkouts' `AGENTS.md` and `.context/`; user guidance in meridian-cli `docs/`. Start with the [system overview](overview.md); terms are in the [vocabulary](vocabulary.md); the [KB guide](AGENTS.md) explains how to maintain these pages.

## Where to look

- **State and history** — [state system](architecture/state-system/overview.md), [native session binding](architecture/native-session-binding.md), [native transcript reads](architecture/native-transcript-reads.md), [portable history](architecture/state-system/portable-history.md), [compatibility between builds](concepts/compatibility.md).
- **Launching and running** — [launch system](architecture/launch-system.md), [session initiation](concepts/session-initiation.md), [continuation model selection](concepts/continuation-model-selection.md), [spawn and chat references](concepts/session-references.md), [process scope](architecture/process-scope.md), [managed primaries](architecture/managed-primary-lifecycle.md), [terminal ownership](architecture/primary-terminal-ownership.md), [session browse](architecture/session-browse.md).
- **Completion and delivery** — [completion hold](architecture/completion-drain.md), [spawn finalization](architecture/spawn-finalization.md), [child result delivery](concepts/child-result-delivery.md), [spawn wait](concepts/spawn-wait-barrier.md), [messaging a live session](architecture/primary-messaging.md), [idle notifications](architecture/idle-notifications.md).
- **Harnesses** — [harness abstraction](concepts/harness-abstraction.md), [Claude](architecture/claude-native-sessions.md), [Pi sessions](architecture/pi-native-sessions.md), [Pi runtime](architecture/pi-runtime.md), [OpenCode](architecture/opencode-harness.md), [Cursor](architecture/cursor-harness.md).
- **Permissions and access** — [tool permissions](architecture/tool-permissions.md), [sandbox projection](architecture/sandbox-projection.md), [workspace](architecture/workspace.md).
- **Mars** — [package management](concepts/package-management.md), [model resolution](concepts/model-resolution.md), [skills](concepts/skills.md), [portable hooks](concepts/package-hooks.md), [launch bundle](architecture/mars-launch-bundle.md), [config precedence](concepts/config-precedence.md).
- **Other mechanisms** — [hooks](concepts/hooks-and-plugins.md), [telemetry](architecture/telemetry.md), [CLI startup](architecture/startup-pipeline.md), 
[per-operation cost](architecture/per-operation-cost.md).
- **Working on the code** — [codebase guide](codebase/guide.md), [releasing](codebase/releasing.md), [test determinism](codebase/test-determinism.md), [lessons](lessons/overview.md).
- **Operations** — [operations](operations/overview.md), including [troubleshooting](operations/troubleshooting.md).
- **Outside evidence** — [research](research/overview.md); **open work** — [open questions](open-questions/overview.md); **related projects** — [ecosystem](ecosystem/overview.md).

The [corrections register](corrections.md) records recurring agent mistakes and their remedies.
