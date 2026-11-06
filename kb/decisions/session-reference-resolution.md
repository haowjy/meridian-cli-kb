# Session Reference Resolution

Tracked session references require an exact recorded native binding. Discovery and primary metadata cannot establish identity: a detected file may belong to another session, and a spawn mirror cannot supply an identity the chat never bound. Continue and fork refuse missing or unbound identities; `--from` can omit an unavailable best-effort pointer.

See [native session identity](native-session-identity.md) and [reference classification](ref-classification.md).
