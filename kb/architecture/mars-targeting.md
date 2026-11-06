# Mars targeting

Mars owns the canonical `.mars/` output and projects skills to harness-native directories. Separate target copies allow harness-specific lowering and avoid duplicate discovery through overlapping shared skill roots.

When Meridian manages routing, native agent emission is suppressed to avoid duplicate agent catalogs; standalone Mars emits native agents so harnesses can discover them. See [package management](../concepts/package-management/overview.md), [package management decisions](../decisions/package-management.md), and the mars-agents checkout's `src/compiler/AGENTS.md`.

