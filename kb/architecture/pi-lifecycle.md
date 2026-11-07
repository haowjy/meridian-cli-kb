# Pi lifecycle and quiescence

A spawned Pi session remains open for follow-up turns while tracked work or delivery is outstanding. Its completion policy requires native settlement, no open compaction, and readable evidence that descendants, managed Bash and owed deliveries have cleared. A turn end alone cannot establish quiescence: a native retry, compaction or delivered result can still require another turn. The source contract is `src/meridian/pi_runtime/.context/CONTEXT.md`.

The shared completion coordinator reconciles persisted descendants; Pi supplies its own native and private-work evidence. Missing or malformed evidence is unknown rather than proof that work is complete. See [completion coordination](completion-drain-coordination.md) and [Pi runtime coordination](pi-runtime/coordination.md).

## Idle done nudge

An idle Pi spawn can receive an advisory done nudge when known execution remains. A done directive can override known running execution, but cannot bypass a result whose publication, admission or public observation is owed. It also waits for an active parent turn and unknown evidence. The nudge is not completion authority.
