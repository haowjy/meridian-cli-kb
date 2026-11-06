# Testing Decisions

The test tiers and no-flaky-tests rule live in meridian-cli `tests/AGENTS.md`.

## D-no-flaky-tests: fix it or delete it

Before deleting a flaky test, break the behavior it protects and run the rest of the suite. If nothing else fails, make its coverage deterministic instead. A flake can expose a product bug, so investigate the product as well as the test. See [test determinism](../codebase/test-determinism.md).

A fake executable must become observable before teardown: process creation does not prove the shim has run. For simple argument logging or fixed output, a POSIX `sh` shim and a bounded wait for its observation avoid racing interpreter startup.
