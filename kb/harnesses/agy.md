# Antigravity (`agy`)

Meridian launches `agy` in no form: the user runs Google Antigravity's own TUI, and Meridian has no `agy` adapter. Antigravity's terms forbid running `agy acp` as a backend worker under a personal Google sign-in, which is the sign-in in use, so spawning `agy` waits until the terms change ([#1016](https://github.com/haowjy/meridian-cli/issues/1016)). Mars carries the same fact: routing never picks `agy`, and Mars writes `.agy` agent files and a host's provided content even when Meridian manages routing ([targets](../mars/package-management.md#targets)).

How `.agy` is laid out and how `agy` treats hooks and MCP servers: mars-agents `docs/config/mcp-and-hooks.md`, "Antigravity". The `agy` behaviours that surprise are under [portable hooks](../mars/package-hooks.md#harness-behavior-to-recheck-after-upgrades).

## Probing `agy`

- Never run `agy acp` or `agy -p` on Jimmy's personal sign-in. Probe through the TUI in a private tmux server (`tmux -L <spawn id>`), with few, tiny typed prompts.
- Run every model turn on `gemini-3.8-flash-low`: launch with `agy --model gemini-3.8-flash-low` and check the TUI shows it.
- Write nothing into `~/.gemini/config/`, `~/.gemini/antigravity-cli/settings.json` or a real repository. A TUI launch adds its own `trustedWorkspaces` entry.
- No sign-ins, no `--remote-control`, and never print tokens.
- Most Mars checks need no prompt: `agy plugin validate --strict` on `.agy`, and the files a sync writes.
