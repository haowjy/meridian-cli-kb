# Jupyter Workbench Runtime Model

A persistent Jupyter kernel captures rich output in real notebooks and retains state across invocations. Visualization runs in that kernel, avoiding a separate daemon and session namespace. Replay is explicit because executing prior cells can be expensive, fail, or change state; a recovered kernel may need visualization setup again.

Append-only working notebooks preserve exploratory provenance. Derivation and compaction can produce a cleaner notebook without erasing that history. Serialized session mutation protects concurrent clients from conflicting writes.
