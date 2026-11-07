# Pi runtime coordination

Pi's managed Bash, spawn watcher and Python observer keep separate evidence of execution, native message admission and public observation. Spawn rows remain the authority for child membership. The source checkout's `src/meridian/pi_runtime/.context/delivery-contract.md` specifies the delivery protocol and its files.

## Child and Bash ownership

A Bash process belongs to its live runtime owner. A cold restart cannot recover process handles from a PID: it records ownership loss and does not signal the old process. Detaching removes tracking, while normal runtime shutdown still stops process groups it owns.

## Completion obligations and delivery

A terminal result remains owed until explicitly consumed or admitted to Pi's native queue. The watcher sends direct child results without consuming them while formatting; a launcher Bash result can transfer its obligation to its canonical children. Unavailable evidence does not establish an empty result.

Native admission and Python's observation are separate facts in separate stores. A crash can occur between them, so a receipt without its matching observation blocks completion with a diagnostic; it cannot prove exactly-once delivery. A cold restart can retry work without a receipt, but must not invent the missing observation. Admission itself does not prove the model acted on a message.

See [Pi lifecycle](../pi-lifecycle.md) and the [delivery decision](../../decisions/pi-runtime.md) for the policy and tradeoffs.
