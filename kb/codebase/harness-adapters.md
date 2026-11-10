# Harness adapters

Adapters translate Meridian's harness-neutral intent into harness-specific
mechanisms. The [harness contract and extension guide](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/harness/.context/CONTEXT.md)
owns the current ports and integration steps; the [harness abstraction](../concepts/harness-abstraction.md)
explains the policy boundary. Adding a harness currently touches more than one
adapter and registration. [`HARNESS_EXTENSION_TOUCHPOINTS`](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/harness/__init__.py)
lists the additional bootstrap, projection, extraction, transport, and accounting
touchpoints; the [cleanup program](cleanup-program.md) tracks the broader leakage.

A parent event scope classifies events rather than filtering them: child events
remain visible to subscribers but must not complete or fail the parent, clear
its signals, or become its report. OpenCode's global stream makes this boundary
particularly important. See [idle adapters](../architecture/idle-adapters.md)
and the [process scope](../architecture/process-scope.md).

OpenCode's managed-primary model selection cannot observe an unsent TUI-local
choice through HTTP, and its GET followed by POST is not atomic against a
concurrent TUI submission. On V1, a `spawn message` follows the last committed
agent rather than a TUI agent switch that has not yet produced a turn. The
[delivery decision](../decisions/primary-inject.md#unified-delivery) explains
why the observer carries messages.

## Claude launch env

Meridian forces a five-minute prompt-cache TTL for non-interactive Claude
spawns. Claude treats `claude -p` as a main conversation, so subscription usage
can otherwise write the higher-priced one-hour cache for a short-lived agent.
The tradeoff is a rebuild after five minutes; no quota saving has been measured.
Claude's settings-file environment can still override the launch environment.
See [prompt caching](https://code.claude.com/docs/en/prompt-caching#choose-the-ttl-yourself)
and the [harness integration guide](https://github.com/haowjy/meridian-cli/blob/main/docs/harness-integration.md).

For native identity and Pi's runtime boundaries, see
[Claude native sessions](../architecture/claude-native-sessions.md),
[Pi native sessions](../architecture/pi-native-sessions.md), and
[Pi runtime coordination](../architecture/pi-runtime/coordination.md).
