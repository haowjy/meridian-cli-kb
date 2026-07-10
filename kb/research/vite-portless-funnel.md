# Research: Vite Host-Check, Portless v0.12.0, and Tailscale Funnel

Synthesized findings from a dedicated external research pass (May 2026) that
validated the dev frontend security decisions DF-D1–DF-D7. See
[architecture/chat/dev-frontend.md](../architecture/chat/dev-frontend.md) for how these
findings shaped the design.

---

## Vite Host-Header Security (CVE-2025-24010)

**CVE:** CVE-2025-24010 / GHSA-vg6x-rcgg-rjx6 (published 2025-01-20)

Vite introduced `server.allowedHosts` in v6.0.9 to guard against DNS rebinding
attacks against the dev server. Prior to that, any page on the local network
could make requests to the Vite server by spoofing the `Host` header.

Key design implications:
- `allowedHosts: true` (blanket wildcard) is **explicitly documented as
  dangerous** — it re-introduces the DNS rebinding attack surface. Never use it.
- Vite skips the host-header check entirely when the dev server is using HTTPS.
  This is the mechanism portless relies on (portless runs Vite under HTTPS by
  default).

**`__VITE_ADDITIONAL_SERVER_ALLOWED_HOSTS` (Vite 6.1.0, PR vitejs/vite#19325)**

Added specifically to support reverse-proxy tools that need to inject allowed
hosts without patching `vite.config.ts`. Portless uses this to add its domain
suffix (e.g. `.<tld>`) to the allowlist. This env var is injected by portless
into its child Vite process after portless starts — Meridian must not clear it
(it is a `__VITE_` var, not a `PORTLESS_` var, so the regex scrub does not
affect it).

**Finding used by:** D5 (allowed-hosts contract)

---

## Portless v0.12.0

**Version verified:** v0.12.0 (released Apr 29, 2026)
**Sources:** `vercel-labs/portless` — `src/{cli.ts,routes.ts,cli-utils.ts,tailscale.ts}`

### `--force` behavior (changed in v0.9.5)

`portless --force` sends SIGTERM to the process that currently owns the
conflicting route, then takes over. This is **not** a lightweight metadata
update — it terminates the existing process. This directly informs why
`--portless-force` must be explicit opt-in (D3).

### PORTLESS_* environment variables (complete set as of v0.12.0)

```
PORTLESS               PORTLESS_PORT          PORTLESS_APP_PORT
PORTLESS_HTTPS         PORTLESS_LAN           PORTLESS_LAN_IP
PORTLESS_TLD           PORTLESS_WILDCARD      PORTLESS_SYNC_HOSTS
PORTLESS_TAILSCALE     PORTLESS_FUNNEL        PORTLESS_STATE_DIR
PORTLESS_URL           PORTLESS_TAILSCALE_URL
PORTLESS_INTERNAL_LAN_IP
```

`PORTLESS_TAILSCALE` and `PORTLESS_FUNNEL` in particular are behavior-changing
vars that would silently override Meridian's intent if not scrubbed. This
validates the blanket-regex scrub approach (DF-D4).

### `__VITE_ADDITIONAL_SERVER_ALLOWED_HOSTS`

Portless auto-injects this Vite env var into child Vite processes. Combined
with Vite's HTTPS host-check skip, portless mode needs no Meridian-side
allowed-hosts derivation (D5).

### Default HTTPS

Portless operates Vite under HTTPS by default (using a local CA). Vite
therefore skips its host-header check, and `VITE_DEV_ALLOWED_HOSTS` is never
needed in portless mode.

**Finding used by:** D3, D4, D5

---

## Tailscale Funnel

**Docs last validated:** Jan 2026. CLI changed in Tailscale v1.52.

### Key facts

- **Beta, disabled by default.** Requires `nodeAttrs: funnel` ACL grant in
  tailnet policy.
- **Prerequisites:** MagicDNS enabled, HTTPS certs enabled, Tailscale v1.38.3+.
- **Allowed Funnel ports:** 443, 8443, 10000 only.
- **Fully public:** No Tailscale auth for external accessors. Anyone with the
  URL can reach the endpoint. This is why `--funnel` requires explicit double
  opt-in (`--funnel` implies `--tailscale`; D2).
- **Non-configurable bandwidth limits** apply.
- `--funnel` correctly implies `--tailscale` in portless; Meridian only needs
  to pass `--tailscale --funnel` to portless (D7).

**Finding used by:** D2, D7

---

## Related Pages

- [architecture/chat/dev-frontend.md](../architecture/chat/dev-frontend.md) — Full design
  incorporating these findings
- [decisions/dev-frontend.md](../decisions/dev-frontend.md) — DF-D1–DF-D7 with rationale
- [tailscale-serve-semantics.md](tailscale-serve-semantics.md) — `tailscale serve`/`funnel`
  routing and teardown behavior (set-path prefix forwarding, portless-443 no-route,
  per-path teardown, MagicDNS lookup) verified during the artifact-serve work
