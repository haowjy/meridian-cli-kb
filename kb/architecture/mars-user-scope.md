# Mars user scope

A user package root supplies agents and skills when no project package root exists, or as a content fallback for a project. Project configuration, aliases, targets and caches remain primary-root-only: a user agent's model alias must therefore resolve in the project using it. The user package root is separate from Mars's user config layer. See [package management decisions](../decisions/package-management.md) and the source checkout's `src/meridian/lib/catalog/AGENTS.md`.

Content root discovery walks upward, while project identity does not; a subdirectory can inherit repository packages but have its own ad-hoc state. See [project identity](state-system/project-identity.md).

The installed Mars beside the running Python takes precedence over `PATH`. Replacing a virtualenv binary in place can alter uv's hard-linked cache; see [dogfooding builds](../lessons/dogfooding-pr-builds.md#writing-over-venvbin-rewrites-uvs-cache).

