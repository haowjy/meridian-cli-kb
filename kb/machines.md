# Saved machines

A session on one machine can read and message a session on another: `meridian spawn list --global --machine <name>`, `meridian session log <machine>/<id>`, `meridian message <machine>/<id> "…"`. The command surface is in meridian-cli `docs/commands.md` ("Machines"); the call and its failure split are in the source checkout's `src/meridian/lib/machines/AGENTS.md`; the identity file, the saved list and their locks are in `src/meridian/lib/state/.context/CONTEXT.md`. This page holds the reasons that span them.

## Plain SSH, peer to peer

The owner chose plain SSH between peers: no Meridian daemon listens for other machines, there is no shared key file, and Tailscale is one way to be reachable, not a requirement. There is no ping and no held-open link, so a call either works or says the machine can't be reached. The model is herdr's saved machines (an SSH target saved under a name and reached with plain `ssh`), copied without depending on herdr. The list of machines lives in Meridian, not in [meridian-comms](ecosystem.md#meridian-comms), which calls the same commands on each saved machine and needs no link of its own.

Each call is a new connection. Measured from a Linux desktop to a Mac mini on the same tailnet, a new connection took 0.22 s and a read about 0.63 s, small against meridian-comms' 30 s poll, so nothing reuses connections. SSH's own reuse (`ControlMaster`) would leave an `ssh` connected after Meridian exits; a user who wants it sets it in `~/.ssh/config`, and Meridian's calls use it.

The other side runs `meridian machine run`, with the request on stdin and nothing the user typed on the remote command line, so no remote shell parses it. It reads the default `~/.meridian`, because a command run over SSH doesn't read a `MERIDIAN_HOME` set in a shell profile.

## Reading, messaging and taking from the inbox cross

The commands that may run on another machine are one list on the command catalog, read by the calling side and again by `machine run` on arrival. `message` delivers, which is what the list is for, and the rest read, with one exception: `inbox take`, `release` and `offer` change the other machine's inbox, because messages filed on the bot's machine have to be taken from sessions elsewhere ([machine inbox](inbox.md#taking-across-machines)). Starting, continuing or cancelling a session on another machine isn't offered: profile, model, sandbox and workspace would all come from the other machine, and the result would have to come back. Printing a machine's name is `machine name`, which may cross; renaming it writes, so it is a separate command, `machine rename`, which may not, because the list holds whole commands and has no rules per argument.

The owner let sandboxed sessions use the crossing commands. A sandboxed session's `meridian spawn` and `meridian message` already run outside its sandbox and can read or message any session on its own machine; reaching another machine with the user's SSH keys adds no new kind of reach, and refusing it would be a check Meridian had to enforce itself.

A command run over SSH starts in the user's home directory, and that directory's project is never the one the caller meant. So a remote command has no current project unless the request names one with `-C`: a [machine-wide ID](launch/session-references.md#ids-are-machine-wide) resolves by its claim, while an old per-project ID or a selector such as `@latest` fails instead of finding a different session in the wrong project. A project is shown beside an ID in listings and message headers, never typed as part of a reference: a project has no name of its own, only a path that differs from machine to machine.

An `@` address doesn't cross. A command arriving from another machine carries no session variables, which meridian-comms reads as a person at a terminal who may post, so a child that may not post could post by going through another machine ([messaging](spawns/messaging.md#people-go-to-a-program-the-user-names)).

## A machine's ID and its name

Every comparison between machines compares a 12-hex-character ID, written once to `~/.meridian/machine.json` and never changed, because a new ID makes every machine that saved this one refuse it. The name is how the ID is written in `<machine>/<id>`. It is computed once from the login and hostname and stored, because on macOS the hostname can change by itself (a Bonjour conflict turns `minibox` into `minibox-2`), and a name recomputed on every call would rename the machine under every peer. After that only `machine rename` changes it, keeping the old names so a reference written with one still means this machine.

Every machine saves another under that machine's own name, with no local label. A message says who sent it as `<machine>/<chat>`, and the receiver replies to that, so both sides have to agree on the name; the ID is what keeps them agreeing after a rename.

The identity has its own file. In `config.toml`, which people sync between machines, it would give two machines one name. In `machines.json`, the saved list a user might copy to another machine, it would travel. Inside `~/.meridian/machine/`, beside the ID claims, it would be lost with that directory, and deleting that directory is meant to be recoverable: the claims are seeded again from the runtime roots, but a lost ID can't be. Because it sits under `get_user_home()`, a different `MERIDIAN_HOME` is a different machine.
