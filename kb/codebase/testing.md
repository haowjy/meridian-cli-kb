# Testing

See the source checkout's `tests/AGENTS.md` and `tests/.context/CONTEXT.md` for test rules and helpers. Synchronize on the condition the assertion observes: persisted state and in-memory API state can lead one another. A runner may still write after its harness exits; wait for every writer before removing a spawn directory. Patches in the test interpreter do not reach child interpreters. A test helper is bound by the same read restrictions as the code it tests. A test mode needs a positive assertion that it reached subprocesses, or a silently disabled mode produces better-looking results; at `sitecustomize` time `sys.argv[0]` is `-m` for `python -m`, so gating instrumentation on the entry-point name misses those children. Running real builds and probes: [verifying changes](verifying-changes.md).

## Race tests must go red against the split implementation

A race test is evidence only if it fails against plausible broken interleavings. A free-running lock/read test missed one broken order in 20 runs; explicitly coordinating the interleaving made the failure reproducible. Use a controlled seam when the race window is too small to hit reliably.

## Cold index builds inside a test deadline: a flake class

Cold index construction once consumed a descendant query's two-second deadline, producing unknown instead of blocked. Build before starting the query deadline; some tests also warm the index to keep setup out of their own wall-clock bounds. See [portable history](../state/portable-history.md#indexes).

Fake time does not eliminate a real deadline in a polling helper; scheduler starvation can still fail the test. Synchronize on the event and investigate product behavior before attributing a timeout to the test.

A process shim must stay alive to receive the input its real counterpart expects; otherwise a connection test races the shim's exit. A fake executable must become observable before teardown, since process creation doesn't prove the shim ran: for argument logging or fixed output, a POSIX `sh` shim and a bounded wait for its observation avoid racing interpreter startup.
