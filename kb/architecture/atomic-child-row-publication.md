# Child rows publish as complete directories

A spawn row is published as a complete directory so readers see either no row or the prompt and state together. Staging sits below a non-row directory so scanners cannot mistake incomplete work for a published child. A store-wide read failure remains unknown and blocks completion; atomic publication does not make failure an empty tree.

See the source checkout's `src/meridian/lib/state/spawn_store.py` and [completion coordination](completion-drain-coordination.md).

