# research/overview — Research Domain

Research pages synthesize external references and explain which ideas Meridian adopted, rejected, or adapted.

## Pages

- [llm-wiki-pattern.md](llm-wiki-pattern.md) — Karpathy's LLM wiki concept and implications for KB structure.
- [tailscale-serve-semantics.md](tailscale-serve-semantics.md) — `tailscale serve`/`funnel` routing and teardown behavior verified live: set-path forwards the full path (no prefix strip), portless 443 does not route local handlers on some nodes, per-path surgical teardown (never bare port off / reset), MagicDNS name lookup, and Funnel prerequisites and port limits. Underpins `meridian artifact serve`.
- [prompt-cache-retention.md](prompt-cache-retention.md) — Idle prompt-cache retention per harness and provider (checked 2026-10-08; Pi providers and ChatGPT backend 2026-10-13); sets the idle-notify TTL defaults and contradicts the spawn-wait yield premise.
- [cursor-sandbox-architecture.md](cursor-sandbox-architecture.md) — Cursor sandbox internals (binary enabled/disabled, `.cursor/sandbox.json`, path control), approval flags (`--force`/`--yolo`), and the mars/meridian mapping decisions.
