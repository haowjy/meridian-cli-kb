# Research: Tailscale Serve & Funnel Routing Semantics

Empirical facts about how `tailscale serve`/`funnel` route requests and how to
tear handlers down safely — verified live on Tailscale 1.98. Anyone building a
serve or reverse-proxy feature over Tailscale hits these walls; they are not in
the CLI's `--help` and contradict naive expectations. Sibling page
[vite-portless-funnel.md](vite-portless-funnel.md) covers Funnel *prerequisites*
(beta, certs, MagicDNS); this page covers the *routing and teardown* behavior.

## Set-path forwards the full path — it does not strip the prefix

`tailscale serve --set-path /<slug> http://127.0.0.1:<port>` forwards the
**entire** request path (`/<slug>/...`) to the backend. The slug is not stripped.
A backend that serves from root (e.g. `python -m http.server`) returns **404**
for `/<slug>/` because no such directory exists.

Backends that serve from root need a **prefix-stripping shim** that rewrites
`/<slug>` → `/` and `/<slug>/<rest>` → `/<rest>` before path translation. The
strip must happen in path translation (not URL rewriting) so directory redirects
keep the prefix. Meridian's `meridian artifact serve` implements this shim.

This is invisible until you fetch the real URL — the serve *configures* cleanly
and the backend works at `/`. Live e2e (curling the public URL), not just
checking the config, is what surfaces it.

## Portless 443 does not route to local serve handlers on some nodes

A `tailscale serve --set-path /<slug> ...` registered on portless 443 (no
`--https=<port>`) can return **404** with an `x-portless: 1` response header.
Tailscale's portless layer does not always route to locally-registered serve
handlers; the same backend on an **explicit** port (`--https=<port>`) works.

Do not assume the portless 443 path routes local handlers. Use an explicit
`--https=<port>` for any local-serve-backed exposure. This rules out the "clean
portless `https://host/<slug>/` URL" design on affected nodes and forces an
explicit port into the public URL.

## Per-path teardown is surgical — never nuke the port or reset

To remove one path handler:

```
tailscale serve --https=<port> --set-path=/<slug> off
```

This removes **only** that slug on that port; sibling slugs and other ports are
untouched.

Never use:
- `tailscale serve --https=<port> off` — drops **all** path handlers on that
  port, including ones you do not own.
- `tailscale serve reset` — wipes **every** serve/funnel on the node. A footgun;
  users routinely have unrelated serves (e.g. a personal 443 → 127.0.0.1:8321).

Only ever remove paths you created. Track the slugs you own and tear those down
individually; let TTL + lazy GC handle expiry.

## Funnel is restricted to ports 443, 8443, 10000

`tailscale funnel` rejects arbitrary ports; only 443, 8443, and 10000 are
Funnel-legal. A random-high-port serve cannot be Funneled. If Funnel exposure is
required, it must target one of those three ports — which conflicts with the
"never claim well-known ports" discipline below. Surface a clear error rather
than silently picking a Funnel-legal port that may clobber an existing 443.

## MagicDNS name

```
tailscale status --json  →  .Self.DNSName  (strip trailing ".")
```

e.g. `pop-os.tail852a76.ts.net.` → `pop-os.tail852a76.ts.net`. This is the
hostname for building shareable `https://<dns>:<port>/<slug>/` URLs.

## How Meridian's artifact serve uses these semantics

The exposure model landed on **explicit random/choosable high port + slug path**
(`https://<host>:<port>/<slug>/`), having tried and abandoned two alternatives:

1. **One random tailnet HTTPS port per serve** — worked, but lost the readable
   shared-443 URL shape and ran into the Funnel-port contradiction.
2. **Portless 443 multiplexed by slug** — broken by the portless-no-route fact
   above (404 with `x-portless: 1`).
3. **Explicit `--https=<port>` + `--set-path /<slug>`** — verified live (200), so
   each serve gets its own random high port (bypasses portless) **and** keeps the
   readable slug. `--port <N>` lets the user choose, rejected if already in use
   or well-known (< 1024).

This aligns with the `structured-artifact` skill's serving guidance: pick a
random or incremented high port — don't claim 443 or other well-known ports.

## Provenance

Live A/B verification on Tailscale 1.98, node `pop-os`, during the
`artifact-serve-cmd` work item (2026-07): reversible tailscale experiments with
state restored identical afterward. Full decision trail in the work item's
`DECISIONS.md`.

## Related Pages

- [vite-portless-funnel.md](vite-portless-funnel.md) — Funnel prerequisites (beta,
  HTTPS certs, MagicDNS, allowed ports) from the dev-frontend/portless angle