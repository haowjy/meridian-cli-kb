# Meridian knowledge base

The KB records cross-cutting intent, rationale, rejected alternatives and external facts that cannot be recovered from the source tree. Directory-specific contracts belong beside the code; user guidance belongs in product docs. Link to those sources rather than copying them.

Write current truth without attribution, work IDs, release status or a history of the page. Retain behavior and the reason for it, project invariants, and external evidence with its sources. Remove inventories and descriptions the code already supplies. Delete pages whose subjects no longer exist; preserve unsettled work in its issue or work directory.

Read the relevant source and existing links before editing. Use relative links within the KB and update inbound links when moving, deleting or changing an anchor. `index.md` catalogs the KB; `decisions.md` indexes decisions. Run `meridian kg check . --strict` after Markdown changes and `meridian mermaid check . --strict` after diagram changes.

The KB checkout is autosynced. Edit it directly unless the work needs an isolated branch. Project-wide invariants live in the source repositories' root `AGENTS.md` files.
