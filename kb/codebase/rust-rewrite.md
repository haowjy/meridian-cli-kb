# The Rust rewrite

meridian-cli is to be rewritten in Rust; the project invariants in the root `AGENTS.md` hold through it. It has no settled design yet. Two problems motivate it:

- **Memory.** The Python import floor is about 90 MB per process, so concurrent Meridian processes multiply the baseline. Algorithmic heap growth is a separate problem that a direct port of the same algorithm would keep.
- **Upgrades crash running processes.** `uv tool install --force` can overwrite modules inside a live process's virtualenv, and a later lazy import then mixes versions ([#628](https://github.com/haowjy/meridian-cli/issues/628)). Until the rewrite, avoid upgrading while long-lived Meridian processes run.

A static binary alone doesn't solve upgrades. Points the design will need to weigh: replacing the binary by rename protects running processes from partial replacement; not re-executing through a mutable path stops a process switching versions mid-run; embedded or versioned assets prevent mixed versions; and versioned shared state lets old and new processes coexist ([compatibility](../state/compatibility.md)). Process-cleanup untangling is deferred to it too ([#940](https://github.com/haowjy/meridian-cli/issues/940)).
