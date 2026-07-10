# research/overview — Research Domain

Research pages synthesize external references and explain which ideas Meridian adopted, rejected, or adapted.

## Pages

- [llm-wiki-pattern.md](llm-wiki-pattern.md) — Karpathy's LLM wiki concept and implications for KB structure.
- [t3-code-reference.md](t3-code-reference.md) — T3 Code architecture analysis and how it informed chat backend choices.
- [vite-portless-funnel.md](vite-portless-funnel.md) — Vite host-check CVE (CVE-2025-24010), portless v0.12.0 behavior (`--force`, PORTLESS_* vars, `__VITE_ADDITIONAL_SERVER_ALLOWED_HOSTS`), and Tailscale Funnel prerequisites. Validates dev frontend decisions DF-D1–DF-D7.
- [tailscale-serve-semantics.md](tailscale-serve-semantics.md) — `tailscale serve`/`funnel` routing and teardown behavior verified live: set-path forwards the full path (no prefix strip), portless 443 does not route local handlers on some nodes, per-path surgical teardown (never bare port off / reset), and MagicDNS name lookup. Underpins `meridian artifact serve`.
- [cursor-sandbox-architecture.md](cursor-sandbox-architecture.md) — Cursor sandbox internals (binary enabled/disabled, `.cursor/sandbox.json`, path control), approval flags (`--force`/`--yolo`), and the mars/meridian mapping decisions.
