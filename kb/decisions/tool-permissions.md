# Profile Tool Permissions

Profiles express tool, sandbox and approval policy; harnesses enforce only what their native mechanisms can express. See meridian-cli `docs/agent-profiles.md` and `src/meridian/lib/safety/.context/CONTEXT.md` for current behavior and gaps.

## Mars owns native names {#d-permission-mars-names}

Mars supplies native permission spellings and the tools they concern. They can differ: Claude scopes a write with an `Edit(path)` permission while the callable tool is `Write`. Deriving one from the other caused silent enforcement gaps.

## Native enforcement {#d-permission-native-enforcement}

Each harness uses its own enforcement mechanism; a universal Meridian interception hook would create a second policy layer that can disagree with the harness. A sandbox is not equivalent to tool denies. A harness without a sandbox reports that gap rather than claiming an approximation. Where a harness has no scoped rule of its own but offers a hook seam, Meridian compiles the profile's rules into that seam: Pi's extension and the [Codex shell hook](#d-codex-shell-hook).

## Codex shell hook {#d-codex-shell-hook}

Scoped `bash` rules bind Codex through a `PreToolUse` hook Meridian owns, because Codex has no scoped shell rule and the profiles use these rules heavily: 184 denies and 34 allows across 7 packaged profiles.

The hook guards against accidents, and it is not a sandbox. The threat is a careless agent running something like `git reset --hard`, never an agent working to get past its own rule. So it splits the command on `;`, `&&`, `||`, `|`, `&` and newlines outside quotes, and matches each segment against wildcard rules with Pi's ranking, where deny wins, so a rule means the same on both. `sudo git reset --hard`, `bash -c`, substitutions, scripts and `git -C` are caught only if a rule matches them. Those are known gaps. Don't close one with a parser: the first version grew a shell parser, wrapper tables, a digest bootstrap and its own deadline, reaching 1,138 lines while still unable to stop a REPL, `apply_patch` or `find -exec`, and wildcard rules cut it to 219 ([#922](https://github.com/haowjy/meridian-cli/pull/922)). The hook's contract and failure handling are in `src/meridian/lib/harness/projections/.context/CONTEXT.md`, its gaps in `docs/upgrading.md`.

Two things stay strict. A profile that refuses what no scoped allow names is an allowlist: every segment must match an allow, and a command holding `$`, backticks or redirection is refused whole. And Codex skips an untrusted hook silently and fails open on hook failures, so Meridian asks `hooks/list` before the first turn and refuses a launch whose hook isn't trusted and enabled, and the hook command turns its own crash into a refusal. Without those a Codex upgrade could turn the rules off with no sign.

## Approval `auto` means the harness's own auto {#d-approval-auto}

`approval: auto` compiles to each harness's own auto: Claude's `--permission-mode auto`, Codex's `on-request` with `approvals_reviewer="auto_review"`. The `acceptEdits` mapping was a stand-in, and leaving the Codex reviewer to `~/.codex/config.toml` made the result depend on the user's home. `--approve-for-me` was rejected because it refuses `--sandbox`, forces `workspace-write` and doesn't exist on the app server. A profile that gates a tool keeps `acceptEdits` or `manual`, whichever makes Claude refuse it, and an allowlist keeps `dontAsk`, because Claude's classifier would otherwise approve the unmatched call and silently widen the profile.

The classifier and reviewer are not security boundaries to rely on. In a live check the Codex reviewer allowed `git reset --hard` and a write to `/etc`, and it never runs under `danger-full-access`. Claude's classifier refused a token exfiltration but not `git reset --hard`. A confined Claude profile under `auto` also lets the classifier decide writes outside the working directories and web tools, including under `network: deny`. The permission report's sandbox and network lines state this, and `docs/upgrading.md` explains it.

## Report actual projection {#d-permission-report-only-gaps}

An enforcement report is produced with the projection so it describes what reached the harness, rather than merely repeating what the profile requested. Unsupported and partial restrictions remain visible in dry run and the projection manifest. The project invariant on lost restrictions in meridian-cli's root `AGENTS.md` governs what shows by default: a lost restriction that changes what an agent can do shows in Mars's default output and in `--dry-run`, not at launch, and every other lost field is available under `--verbose`.

## Primaries {#d-permission-primaries-bound}

Profile rules bind primaries as well as spawns. The human can widen their own interactive session, and native user settings or passthrough flags can affect enforcement; these are reported rather than stripped.

## D-permission-headless-ask: `ask` in a spawn is refused at once {#d-permission-headless-ask}

A spawn has no human to answer, so a supported `ask` is
refused immediately with a reason the model can read, recorded in the spawn's
`permission_requests.jsonl`, and listed by `meridian spawn show`
(`denied_requests` in `--json`). The request is not routed to the parent agent. A primary keeps `ask`. Claude spawns disable permission prompts; Pi's extension
blocks with Meridian's reason. Approval requests a harness raises from its own
approval mode are refused and journaled the same way, which needed the Codex
reply `decline` and the OpenCode V2 reply key `decision`.

> [!FLAG] **Needs human review.** The OpenCode divergence below remains open for review.

**Accepted divergence (OpenCode):** the native mechanism lowers `ask` to native
`deny` in OpenCode spawns. So the model sees OpenCode's own refusal text, not
Meridian's reason, and a tool native policy removes from the catalog produces no
attempted call to journal. Forcing Meridian's reason would need a mediation hook,
which D-permission-native-enforcement rejects, or an upstream seam for custom
denial reasons. Tracked in meridian-cli `src/meridian/lib/harness/.context/TODO`.

## Scoped allowlists {#d-permission-allowlists}

A capability with only scoped allows is an allowlist for that capability; approval policy does not turn unmatched calls into allows. Harnesses vary in what they can enforce. Claude's deny-before-allow ordering and automatic approvals make this particularly subtle; consult the projection report and current harness context.

## List form {#d-permission-list-form-deprecated}

A tools list grants named tools; it does not mean only those tools. The form is deprecated because Claude agent lists and skill allowed-tools have different meanings. Use a map with `'*': deny` for a whole-profile allowlist. Imported Claude agent lists are lifted with their restrictive meaning intact.

## Permission set merging {#d-permission-set-merge}

Tool grants and denies and network host sets add across surfaces; an explicit replace form is available. Ordered and keyed lists keep their own replacement semantics. A package profile cannot replace a consumer's deny set: that would let a dependency erase the consumer's restriction.

## Platform rules {#d-permission-platform-rules}

Meridian adds the control-plane grants and delegation restrictions its own operation needs; the native projections and limitations are in [sandbox platform access](sandbox-platform-access.md). A completion command must be available even under an allowlist or a child cannot signal completion.

## Unset sandbox {#d-codex-sandbox-inference-deleted}

An unset sandbox means the harness default. Inferring it from the tool list conflates separate controls.

## User settings {#d-claude-settings-files-kept}

Claude's user, project and local settings stay in force; their allows may widen a profile's allowlist. Removing those sources would also discard their hooks. The report exposes this limitation.

## Open questions

Mars still reports a Codex profile's scoped `bash` rules as lost. The hook now binds them on a Meridian launch, but a Codex agent run natively has no hook, so whether that line can go quiet is unsettled.

A Codex profile with no sandbox can inherit Codex's read-only default without Meridian knowing it; completion confirmation may then lack a writable signal channel. Explicit read-only policy lets Meridian handle that case.
