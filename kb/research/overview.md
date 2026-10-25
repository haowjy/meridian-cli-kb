# research/overview — Research Domain

Research pages synthesize external references and explain which ideas Meridian adopted, rejected, or adapted.

## Pages

- [llm-wiki-pattern.md](llm-wiki-pattern.md) — Karpathy's LLM wiki concept and implications for KB structure.
- [tailscale-serve-semantics.md](tailscale-serve-semantics.md) — `tailscale serve`/`funnel` routing and teardown behavior verified live: set-path forwards the full path (no prefix strip), portless 443 does not route local handlers on some nodes, per-path surgical teardown (never bare port off / reset), MagicDNS name lookup, and Funnel prerequisites and port limits. Underpins `meridian artifact serve`.
- [prompt-cache-retention.md](prompt-cache-retention.md) — retention by harness, provider and authentication; signed-in Codex versus API-key policy, the separate #707 idle-warning divergence, compaction cache reads and Pi cache warming.
- [cursor-sandbox-architecture.md](cursor-sandbox-architecture.md) — Cursor sandbox internals (binary enabled/disabled, `.cursor/sandbox.json`, path control), approval flags (`--force`/`--yolo`), and the mars/meridian mapping decisions.
- [claude-code-mods-and-inbox.md](claude-code-mods-and-inbox.md) — What Claude Code 2.1.302 on Linux does with outside input: mods under `-p` and in a TUI, and the cross-session inbox socket with its inbound controls; later rejected-surface probes characterize `$.ui.status` rows through 2.1.306. Underpins Claude children keeping stdin and Claude primary inject.
- [harness-plugin-loading.md](harness-plugin-loading.md) — How Claude Code, Codex, OpenCode and Cursor load plugins, probed: one Claude-format tree serves Claude and Codex; Codex loads no plugin agents; Claude session agents don't preload `skills:`. Underpins `mars build plugin`.
- [opencode-config-semantics.md](opencode-config-semantics.md) — OpenCode 2.0.29 and 1.19.3 prompt, agent and model channels verified live: which config keys each version ignores, `{file:}` substitution rules, `default_agent` scope, v2 CLI surfaces. Underpins OpenCode prompt delivery.
