# Decisions: Managed Command References

Decisions about how mars-agents surfaces command references to users when running standalone (`mars`) versus through Meridian (`meridian mars`).

For mechanism, see:
- [ecosystem/overview.md](../ecosystem/overview.md) — mars-agents as Meridian's package manager backend
- [decisions/package-management.md](package-management.md) — Mars compiler, sync, and targeting architecture

---

## `managed_cmd()` — MERIDIAN_MANAGED-aware command references (D76)

### Problem

mars-agents had ~20 hardcoded `` `mars <cmd>` `` strings scattered through user-facing output: sync hints, error messages, doctor output, and repair suggestions. When invoked through Meridian (`meridian mars sync`), users saw `mars sync` instead of `meridian mars sync` — confusing because `mars` is not in the user's PATH in that context, and because running a naked `mars` command bypasses Meridian's environment setup.

Suppressing hints entirely was considered and rejected early (the user called it "stupid" — they want correct command references, not absent ones).

### Solution

Added `managed_cmd(cmd: &str) -> Cow<'_, str>` to `mars-agents/src/types.rs`:

- Returns `"meridian mars <cmd>"` when `MERIDIAN_MANAGED=1` is set in the environment
- Returns `"mars <cmd>"` otherwise
- Uses `std::sync::OnceLock<bool>` to read the env var exactly once per process (thread-safe, zero cost on repeated calls)
- Returns `Cow::Borrowed` in the standalone case (no allocation)

Every user-facing string that named a mars command was updated to call `managed_cmd()` instead of embedding a literal.

**Corresponding meridian-cli changes:** Two messages in the Python codebase also required updating — the catalog startup redirect message (`src/meridian/cli/startup/catalog.py`) and a doctor warning (`src/meridian/lib/ops/diag.py`). These had the inverse problem: they hardcoded `meridian mars` when they should have reflected the actual invocation. Fixed to use the correct context-aware string.

### Files changed (mars-agents)

| File | Change |
|---|---|
| `src/types.rs` | New `managed_cmd()` helper |
| `src/sync/mod.rs` | Sync hint (`upgrade --bump`) |
| `src/cli/output.rs` | Sync and conflicts hints |
| `src/cli/doctor.rs` | Three doctor messages |
| `src/cli/link.rs` | `mars init` hint |
| `src/cli/mod.rs` | `mars repair` hint |
| `src/target_sync/mod.rs` | Sync and repair hints |
| `src/error.rs` | `mars init` and `mars models refresh` error messages |
| `src/config/mod.rs` | Deprecated `.agents` hint |

### Files changed (meridian-cli)

| File | Change |
|---|---|
| `src/meridian/cli/startup/catalog.py` | Redirect message |
| `src/meridian/lib/ops/diag.py` | Doctor warning message |
| `tests/integration/ops/test_diag.py` | Updated assertion to match new message |

### Alternatives rejected

**Suppress/hide hints when `MERIDIAN_MANAGED=1`** — Rejected. The user explicitly asked for correct references. Absence is worse than presence; a user who gets no hint at all is more confused than one who gets the wrong binary name.

**Full i18n / message catalog framework** — Far-future work, not worth the complexity for a naming concern. The problem is one of context-aware naming, not localization.

**String macros** — Harder to maintain and grep. A plain function call is more readable and no less efficient given `OnceLock`.

### Design note: `Cow<'_, str>`

The `Cow` return type matters. In standalone mode, `managed_cmd()` returns a `&'static str` wrapped in `Cow::Borrowed` — no heap allocation. Only the managed path allocates (one `format!` call per invocation, cached by `OnceLock`). This keeps the standalone hot path zero-cost and avoids changing callers' type signatures in most cases.

---

## Related

- [decisions/package-management.md](package-management.md) — sync model, upgrade semantics
- [ecosystem/overview.md](../ecosystem/overview.md) — mars-agents and Meridian relationship
