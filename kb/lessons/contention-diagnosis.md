# Contention and Cost Diagnosis

A deadline error can mean a lock holder is slow, not that the deadline is too short. Measure the uncontended operation and identify what runs under each lock before adding retries. A held-lock reproduction tests recovery but does not identify the holder. Pending-source labels may name bystanders rather than the lock that blocked progress.

## Check a cause ranking against the protocol

Trace who waits on each candidate and what they hold while waiting. A marker that stays pending under churn can make readers wait for later writers, even if the expensive work runs before the writer marks it.

## Count work; don't time it on a shared box

Count physical reads, scans, and lock acquisitions per complete command to find scaling. Capture call stacks for unexplained counts. Run the command at several history sizes: timing isolated seams misses work elsewhere in the path.

## Count every pass per operation, not one kind of fold

Count filtered and unfiltered passes together. A replacement can eliminate one kind of scan while multiplying another. For fan-out operations compare wall time and lock hold as well as counts.

Warm a copied index before measuring warm behavior; report cold build separately. Compare old and new on the same corpus under comparable load, alternating runs, and report correctness alongside timing. See [per-operation cost](../architecture/per-operation-cost.md) and [history storage](../architecture/state-system/portable-history.md).
