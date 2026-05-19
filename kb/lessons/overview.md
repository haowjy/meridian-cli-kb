# lessons/overview — Lessons Domain

Lessons pages preserve hard-won context from failures, abandoned approaches, and integration surprises.

## Pages

- [state-design-lessons.md](state-design-lessons.md) — Why dual-root and JSONL, earlier failures, and state-design tradeoffs.
- [harness-integration.md](harness-integration.md) — Claude, Codex, and OpenCode integration surprises and generalized patterns.
- [chat-normalization-repair.md](chat-normalization-repair.md) — Harness protocol drift lessons, completion-dedupe rules, replay implications, and live smoke cautions from the 2026 chat repair.
- [mars-compiler-cleanup.md](mars-compiler-cleanup.md) — Mars cleanup lessons: Windows config artifacts, lock indexing, test splitting, dead-code deletion, and warning routing.
- [mars-launch-bundle-lessons.md](mars-launch-bundle-lessons.md) — Mars/Meridian launch-bundle lessons: schema split discipline, contract-bound test splits, OpenCode env merging, and experimental Cursor signaling.
- [source-simplification.md](source-simplification.md) — Lessons from Phase 8.6 source-seam and test-collapse work: deletion-first simplification, test contract discipline, seam ownership moves, and over-collapse recovery.
- [arch-refactor-pitfalls.md](arch-refactor-pitfalls.md) — Architecture-refactor implementation pitfalls: git-hook environment leaks, structlog capture blind spots, checkpoint staging scope, process-group cleanup, Windows asyncio signal handler silently dropped, Windows Claude cancel SIGINT no-op, audit false-positive validation discipline, and no-arg spawn wait full-chat-tree bug.
- [pi-rpc-quiescence-impl.md](pi-rpc-quiescence-impl.md) — Pi RPC quiescence implementation lessons: Windows path escaping in JSON fixtures, Node ESM file:// URLs, pnpm version mismatch (10 vs 11), lockfile discipline, flaky test design flaws, batching CI failures, platform-aware path assertions, and cross-platform filesystem error behavior.
