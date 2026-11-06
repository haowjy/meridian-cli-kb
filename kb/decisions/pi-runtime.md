# Pi Runtime Ownership and Delivery

Current contracts are in meridian-cli `src/meridian/pi_runtime/.context/CONTEXT.md` and `delivery-contract.md`.

## Live execution ownership

A managed-Bash owner retains process handles across extension reload. After a cold restart, a recorded PID cannot prove ownership: it may have been reused, so recovery marks ownership lost and never signals it. A failed advisory ping releases its claim without killing the shell; a failed execution or durable publication is a task failure.

## Result delivery evidence

A queued message does not prove native admission. Exact result membership, native admission, and Python's public-event observation are separate evidence. Only canonical direct-child rows establish membership; a Bash origin transfers an obligation but cannot create a child relationship. A receipt without its matching observed event is unknown and eventually fails closed. These stores cannot commit atomically across a crash, so exactly-once delivery is not guaranteed.

Automatic notices are bounded in aggregate as well as per item: otherwise a large backlog can exhaust file descriptors or let one large report starve others. Each included item keeps its identity, outcome and a way to read the full result; unrepresented work remains owed. A successful tool boundary can deliver completed work while the parent is active; idle delivery handles genuinely late work. Explicit waits can consume a result first.

A queued message Pi never admits may keep IDs reserved and pause unrelated publication until a native lifecycle callback, reload or restart. This is an open limit, not proof of a reproduced defect; see [future work](../open-questions/future-work.md#pi-result-delivery-limits).
