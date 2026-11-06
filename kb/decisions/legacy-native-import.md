# Legacy Native Session Import

The one-time import binds old chats only when recorded identity and a matching native store provide an unambiguous key. Damaged sources defer the import rather than permanently recording a missing session. See [legacy native import](../architecture/legacy-native-import.md) for the mechanism.

## Old Pi chats that 0.6.7 never bound: content-proven recovery, manual repair

Old Pi spawns sometimes have no recorded native ID. Automatic recovery requires a unique candidate in the spawn's own directory and matching retained prompt or report content. A unique candidate with matching cwd and time alone once selected the wrong session; bindings are immutable. Primaries used a shared directory and are not bound automatically. Manual `session repair` provides a user-verified path for ambiguous cases. A project-local spawn ID alone cannot prove ownership of a user-global Pi directory.
