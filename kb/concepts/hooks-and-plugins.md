# Hooks and Plugins

Hooks observe lifecycle and notification events for side effects. They are fail-open: a failure is reported but does not stop the lifecycle or later hooks. Hook rows and event behavior are documented in `src/meridian/lib/hooks/AGENTS.md` and `docs/`.

`meridian hooks list`, `check`, and `run` use the actual cwd rather than an inherited project environment, because a subprocess may be working in another checkout. Notification emission instead follows the session project, so the two can intentionally differ.

Git autosync is a hook rather than a state authority. See [autosync decisions](../decisions/state.md) and the source guidance under `src/meridian/lib/hooks/builtin/`.
