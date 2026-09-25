# Decision: Pin each chat to one native session; wrap native transcripts

**Status: settled 2026-09-24.** Phase 1 (immutable binding + Pi exact identity) is
implemented and reviewed on `fix/native-session-wrapper`, not yet merged. The
Claude/Codex/OpenCode exact-identity lane is in progress. Pi exit observation, native
readers, and runner-history removal have not started. How the seams work:
[native session binding](../architecture/native-session-binding.md); Pi specifics:
[Pi native sessions](../architecture/pi-native-sessions.md).

## Decision

A Meridian chat `cN` is permanently associated with one **native key**:
`(harness, native_store, native_session_id)`. The store is part of the key because
the same ID in a different directory or database is a different conversation. A chat
is not a run, a retry, a transcript snapshot, or a pointer to "whatever the harness
has open now."

- **`--continue cN` resumes exactly that key or fails.** Failure is typed:
  `unbound` (the chat never acquired a key) or `missing` (the key's transcript is gone
  or not yet persisted). There is no fallback to another candidate ID, primary
  metadata, an adapter scan, or runner history. A tracked reference without a recorded
  harness refuses instead of guessing one.
- **Exit may land elsewhere; the source is never repointed.** Inside a harness TUI the
  user can `/resume` or `/new`. If a run enters on c5/X and exits on Y, Y maps to its
  own existing or new chat; c5 stays X. An exit Meridian cannot observe is
  `unresolved`, and the chat stays at its entry key.
- **Meridian is a wrapper over harness-native transcripts.** The native journal is the
  conversation. Meridian's own `spawns/<id>/history.jsonl` runner stream is a second
  copy that creates a second candidate authority. New runner-stream writes will stop,
  and existing files stay as labeled legacy evidence. SQLite stays a disposable
  search/preview projection, never binding or transcript authority. Removing the
  runner stream comes after native readers. It has not happened yet.
- **Not a new initiation mode.** `--from` (fresh session plus lightweight context),
  `--fork`/`--fork-fresh`, `-f`, and `spawn inject` keep their meanings. See
  [session initiation](../concepts/session-initiation.md).

## Entry evidence is a harness-guaranteed exact target

Before exec, Meridian decides the exact native target and binds it to the chat. The
target is one of:

| Operation | What Meridian does before exec | Harness guarantee relied on |
|---|---|---|
| Create | Mints the ID (Pi, Claude) or obtains it from an owned pre-input response (Codex `thread/start`, OpenCode `POST /session`) | Harness creates or reopens exactly that ID |
| Resume | Verifies the exact locator (file exists, header ID equals the key, store matches) | Harness opens that path/ID as-is |
| Fork | Verifies the source; assigns the new target ID where the harness accepts one | Harness writes a new session that records its parent |

The binding happens under the sessions lock and is bind-once. The first owned
identity signal the harness emits (Pi `session_start`, Codex/OpenCode API response,
Claude hook or connection ID) confirms it. If that signal contradicts the target, the
attempt fails as `entry_mismatch`. No binding is created for the unexpected key, and
no chat's key changes. Identity switches after that are ordinary switches, not entry
evidence.

**Why argv counts as evidence here.** The earlier rule was "planned argv is not an entry
binding". It demanded an *observed* identity before any input reached the model. On Pi
that meant an owned pre-input gate, which exists only in `--mode rpc`. Meridian has no
RPC frontend for the interactive TUI that people actually use, so primary Pi could
never be tracked under that rule. The comparison branch that built toward it (PR #519,
~22k lines: RPC-primary route, raw-argument grammars R1/R2, model-intent fold C1–C3,
owner/coordinator layers, `transport_unqualified` refusal) never enabled a single
tracked continue. The correction is to ask what the harness itself guarantees. When
Meridian verified the target and the harness contract says it opens exactly that
target, the emitted operation is evidence. Anything the contract does not cover is
disclosed as a limit instead of being hidden behind more machinery.

Some pieces of that branch were kept: the Pi session-boundary extension (for exit
observation), the Pi reopen-default lineage projector (for native reads), and refusal
of raw passthrough flags that would override managed identity.

## Discovery is never identity evidence

Choosing a journal by cwd, mtime, newest file, or ID prefix is never identity
evidence. Incident p6615 is the reason. A fresh Pi primary had no assigned ID, so
Meridian took the newest same-cwd journal in a shared directory, and a concurrent
same-cwd session could be selected. The wrong ID was persisted as canonical, and a
later resume loaded someone else's conversation. That was real model-context
contamination, not a display bug.

Status: Pi discovery is deleted. Claude, Codex, and OpenCode still have
filesystem observation legs (Codex rollout scan, OpenCode storage/log detection,
Claude exact-ID lookup across config roots). Their lane is in progress. Those legs can
still supply a *first* observation for an unbound chat, but the bind-once seam stops
them from overwriting a key. Claude's TUI trampoline successor is now a logged
conflict. It is never adopted, because a same-project prompt match is inference, not
an owned signal.

## Rejected alternatives

- **Observed-entry-only rule with a Pi RPC primary.** Rejected for the reasons above.
- **Repointing a chat on switch**, or treating every switch as fatal. Repointing
  corrupts `--continue`. A fatal switch breaks normal TUI use.
- **Mirrors as authority.** Primary metadata, spawn-row IDs, and multi-ID session
  arrays could each "recover" an identity. The chat binding is the only authority;
  mirrors copy the accepted ID.
- **A separate file-pin registry or conflict journal.** Resume re-resolves the exact
  file inside the recorded store and verifies its header. The store is already in the
  key.
- **Fail-closed UUID minting on unreadable siblings.** A single torn journal in the
  shared primary store would block every fresh launch. Minting skips unreadable
  headers with a warning. Source resolution and post-exit verification stay
  fail-closed.

## Accepted limits

- External deletion or replacement of a verified file between preflight and the
  harness opening it is outside the guarantee. Pi, for example, initializes a new ID at
  a missing path. Detection fails the attempt, but input may already have reached the
  model.
- Collision checks rule out an existing ID when they run. They are not an atomic
  reservation against external writers. The protection is UUID4 improbability plus
  the check.
- Model/provider selection on reopen is a separate concern and never changes identity.

## Phases

| Phase | Scope | State |
|---|---|---|
| 1 | Immutable binding, pre-exec plan seam, exact-only resolution, Pi exact identity; Claude/Codex/OpenCode exact identity | Pi + core done and reviewed; other harnesses in progress |
| 2 | Pi exit observation via session-boundary extension; B→own cN; isolated real-Pi 0.87.1 qualification | Not started |
| 3 | Native readers: `session log`/context/search on the exact key; Pi reopen-lineage view | Not started |
| 4 | Remove runner-history writers/readers/checkpoints; measure cost | Not started |

Phase 1 was verified with fake `sh` harness shims at the real launch seams and with
CLI probes against an isolated built wheel. It is **not runtime-qualified against a
real Pi binary**. That qualification is in phase 2.

**Provenance:** `work:native-harness-session-identity` (`decision.md`,
`DIVERGENCE/exact-locator-entry.md`, `design/native-history-only.md`); design review
`spawn:p7037`; lanes `spawn:p7038`, `spawn:p7040`, `spawn:p7043`, `spawn:p7048`; reviews
`spawn:p7039`, `spawn:p7044`, `spawn:p7045`; incident `spawn:p6615`.
