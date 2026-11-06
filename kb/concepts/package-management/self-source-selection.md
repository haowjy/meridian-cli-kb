# Current-Package Source Selection

Mars selects current-package content under a synthetic self owner. `.mars-src` overrides matching declared-package definitions before staging; dependencies are renamed before self items overlay installed destinations. This preserves dependency naming and references when a dependency's authored name differs from its installed name. See `src/sync/AGENTS.md` in mars-agents.

Discovery chooses the shallowest occupied convention layer. Generated non-hidden target content can therefore become source on a later sync; [mars-agents #161](https://github.com/haowjy/mars-agents/issues/161) tracks the general case. Plugin output is excluded through its sentinel so a prior build cannot shadow authored package content.

An unowned destination is not adopted merely because its bytes match. Explicit force adoption is limited to safe shapes; if a later apply fails, that adoption is not journaled and an ordinary retry may still refuse the unowned output. See [sync model](sync-model.md) and `src/sync/AGENTS.md`.

Canonical write recovery does not cover native and config outputs; [mars-agents #149](https://github.com/haowjy/mars-agents/issues/149) tracks that boundary.
