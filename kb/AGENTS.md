# Meridian knowledge base

The KB holds what spans more than one directory and can't be read from the source: cross-cutting intent, the reason behind a behaviour, external facts about harnesses and upstream tools, and open questions. Directory contracts live beside the code in `AGENTS.md` and `.context/`; user guidance lives in `docs/`. Link to those instead of copying them.

## Writing

- State the behaviour and its reason as current truth: no attribution, names, session or spawn ids, release status, or history of the page.
- Keep it short. Don't describe what the code already shows, and don't keep inventories of files, functions or fields. Anything cut can be researched again or asked of the owner.
- A rule whose reason nobody can state gets deleted, not kept. Rules without reasons get turned into code nobody can justify.
- Only what the owner chose, or what shipped, belongs here. An option they floated or a question they asked stays in the work directory until they choose.
- Write by topic: no decision ids, and no list of rejected options. Keep a rejected option only with the fact that ruled it out.
- Project invariants live in the root `AGENTS.md` of meridian-cli and mars-agents. Everything here is a current choice, open to rewriting when evidence shows something better.
- Delete a page whose subject no longer exists.

## Structure

`index.md` is a short map of the directories and the pages to read first. A topic page holds the mechanism and the reasons behind it together; `architecture/idle-notifications.md` is the model. Pages under `decisions/` still hold reasons apart from their topic: when you edit one, merge it into its topic page and point its links there. A topic gets its own directory once it has several pages.

## Landing changes

Edit the autosynced checkout (`$MERIDIAN_CONTEXT_KB_DIR`) directly; the autosync hook commits and pushes it. Use a branch (a worktree of the KB repo) only for a large update you don't want landed until it's finished. Merge it without asking once it's ready.

## Checks

Use relative links, and fix inbound links when you move or delete a page or change an anchor. Before finishing, run `meridian kg check . --strict`, plus `meridian mermaid check . --strict` after diagram changes.
