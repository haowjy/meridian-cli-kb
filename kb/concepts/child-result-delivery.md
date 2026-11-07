# Child result delivery

Background child results reach a lead through `spawn wait` or, for supported Claude and Pi sessions, automatic delivery. The fallback can wake a lead that has already ended its turn; instructions to wait cannot. See [wait](spawn-wait-barrier.md) and [delivery decisions](../decisions/child-result-delivery.md).

A lead is owed its direct background children's unconsumed launches. Continuation can produce another owed launch under the same spawn ID. A nested lead owns its own children, not its siblings or its parent's backlog. The source checkout's [src/meridian/lib/ops/spawn/.context/CONTEXT.md](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/lib/ops/spawn/.context/CONTEXT.md) and [src/meridian/pi_runtime/.context/delivery-contract.md](https://github.com/haowjy/meridian-cli/blob/main/src/meridian/pi_runtime/.context/delivery-contract.md) own the contracts.

Interactive Claude primaries and Pi sessions can receive automatic delivery; spawned Claude, Codex and OpenCode leads must wait explicitly. Claude sends headers and acknowledges only after the delivered turn starts: submission alone proves the prompt was queued, not seen. Pi delivers bounded report excerpts; whether it should also use headers remains open in [delivery decisions](../decisions/child-result-delivery.md).

## Consumption sources

Delivery considers command observations, Pi receipts and dismissals, including legacy records. A wait lease prevents a watcher from competing with a live wait. Observation follows the surface that actually shows a result; internal reads do not imply delivery. Recording is best effort, so a failed observation can cause a duplicate.

## Legacy ledger files

Launch-keyed records use new file names so older builds cannot overwrite a format they do not understand. An older Python reader treats an unknown version as absent and rewrites it; an older Pi reader stops delivery. Older builds may instead redeliver a result consumed by a newer build. See [spawn launches](../architecture/state-system/spawn-launches.md).

## Accepted duplicates and losses

Delivery is best effort. A wait records consumption before its output is printed, so a process killed between those steps can lose delivery. A collect racing a wait can deliver a duplicate. These windows avoid a separate claim protocol; details and alternatives are in [delivery decisions](../decisions/child-result-delivery.md).
