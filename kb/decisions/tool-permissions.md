# Profile Tool Permissions

Profiles express tool, sandbox and approval policy; harnesses enforce only what their native mechanisms can express. See meridian-cli `docs/agent-profiles.md` and `src/meridian/lib/safety/.context/CONTEXT.md` for current behavior and gaps.

## Mars owns native names {#d-permission-mars-names}

Mars supplies native permission spellings and the tools they concern. They can differ: Claude scopes a write with an `Edit(path)` permission while the callable tool is `Write`. Deriving one from the other caused silent enforcement gaps.

## Native enforcement {#d-permission-native-enforcement}

Each harness uses its own enforcement mechanism; a universal Meridian interception hook would create a second policy layer that can disagree with the harness. A sandbox is not equivalent to tool denies. A harness without a sandbox reports that gap rather than claiming an approximation.

## Report actual projection {#d-permission-report-only-gaps}

An enforcement report is produced with the projection so it describes what reached the harness, rather than merely repeating what the profile requested. Unsupported and partial restrictions remain visible in dry run and the projection manifest.

## Primaries {#d-permission-primaries-bound}

Profile rules bind primaries as well as spawns. The human can widen their own interactive session, and native user settings or passthrough flags can affect enforcement; these are reported rather than stripped.

## D-permission-headless-ask: `ask` in a spawn is refused at once {#d-permission-headless-ask}

**Decided (owner):** a spawn has no human to answer, so a supported `ask` is
refused immediately with a reason the model can read, recorded in the spawn's
`permission_requests.jsonl`, and listed by `meridian spawn show`
(`denied_requests` in `--json`). The owner chose "refuse and log it"; never
routing the request to the parent agent was part of the recommendation the
owner accepted. A primary keeps `ask`. Claude spawns disable permission prompts; Pi's extension
blocks with Meridian's reason. Approval requests a harness raises from its own
approval mode are refused and journaled the same way, which needed the Codex
reply `decline` and the OpenCode V2 reply key `decision`.

> [!FLAG] **Needs human review.** The lead accepted the OpenCode divergence
> below without an owner ruling; the owner was told and has not answered.

**Accepted divergence (OpenCode):** the owner mechanism lowers `ask` to native
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

A Codex profile with no sandbox can inherit Codex's read-only default without Meridian knowing it; completion confirmation may then lack a writable signal channel. Explicit read-only policy lets Meridian handle that case.
