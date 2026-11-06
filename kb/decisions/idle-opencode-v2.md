# OpenCode 2.x Idle Sensor

## OpenCode 2.x sensor {#d-opencode-idle-v2--opencode-2x-gets-its-own-sensor-chosen-by-the-resolved-transport}

The sensor follows the transport's resolved dialect: the configured preference can be `auto`, and resolving it independently could disagree with the connection. OpenCode 2.x has different turn and inbox events from 1.x. A user inbox item counts as a return both on enqueue and delivery: a steered prompt may run within the current execution, while a queued prompt can arrive after an idle stretch has started.

A managed primary reconnects its event stream for its lifetime; a spawn instead reconciles a stalled stream and terminates. Treating the primary like a spawn leaves later turns unobserved. See [idle adapter architecture](../architecture/idle-adapters.md).
