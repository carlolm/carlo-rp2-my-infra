# carlo-rp2-my Internal Infrastructure

Canonical config **and** runbook for the internal web apps on `carlo-rp2-my`,
fronted by Caddy and gated behind Google OAuth2. **This repo is the source of
truth** — deploy by pushing here and pulling on rp2; never edit in place.

## Deployment model

rp2 runs everything as **native systemd services** using single binaries in
`~/bin` — **not** Docker (Docker isn't installed on rp2). Three units, all
running as user `carlo`, reading config from this repo at
`/home/carlo/carlo-rp2-my-infra`:

| Unit | Binary | Reads |
|---|---|---|
| `gatekeeper-oauth.service` | `~/bin/oauth2-proxy` | `oauth2-proxy.cfg` (gitignored — has secrets) |
| `gatekeeper-caddy.service` | `~/bin/caddy` | `Caddyfile` |
| `gatekeeper-tunnel.service` | `~/bin/cloudflared` | `TUNNEL_TOKEN` from `.env` |

Reference copies of the unit files live in [`systemd/`](systemd/).

- **Machine:** `carlo-rp2-my` — Tailscale IP `100.111.137.61`
- **Repo on rp2:** `/home/carlo/carlo-rp2-my-infra`
- **Local working dir (Mac):** `~/dev/carlo-rp2-my-infra`
- **Deploy flow:** edit on Mac → commit → push → `git pull` on rp2 →
  `sudo systemctl restart gatekeeper-oauth gatekeeper-caddy` (restart only what changed).

## Apps & ports

Upstreams are local services on rp2. Ports are authoritative from the `Caddyfile`.
**Access** shows how each is reachable:

| App | URL | Upstream | Access |
|---|---|---|---|
| Gatekeeper (OAuth2) | https://auth.carlolm.com | `127.0.0.1:4180` | **Public** (tunnel) — login endpoint |
| Nutrition (frontend) | https://nutrition.carlolm.com | `127.0.0.1:7200` | **Public** (tunnel), Google-gated |
| Nutrition (API) | `nutrition.carlolm.com/api/*` | `127.0.0.1:7201` | **Public** via same-origin `/api/*`, gated |
| Nutrition (API, direct) | https://api-nutrition.carlolm.com | `127.0.0.1:7201` | Tailscale-only (script alias) |
| Infra Dashboard | https://infra.carlolm.com | `127.0.0.1:7100` | Tailscale-only |
| Newsletters | https://news.carlolm.com | `127.0.0.1:7300` | Tailscale-only |

## Architecture

- **Caddy** — reverse proxy + automatic HTTPS via the Cloudflare **DNS-01**
  challenge (no public ports needed for certs). Each gated site imports the
  `(gated)` snippet, which `forward_auth`s to oauth2-proxy and redirects
  unauthenticated requests to the login endpoint.
- **OAuth2 Proxy** — the "Gatekeeper." Forces Google login; only addresses in
  `allowed_emails.txt` pass. Cookie domain `.carlolm.com`, so one login covers
  every subdomain.
- **cloudflared** — runs the remotely-managed `rp2-apps` Cloudflare Tunnel for
  **public** ingress. Public-hostname routing is configured in the Cloudflare
  dashboard (not in this repo). Routes hit Caddy locally at `https://localhost:443`,
  so public traffic still passes through the oauth2-proxy gate.
  - Each public route needs **Origin Server Name** = its hostname (so cloudflared
    sends the right SNI; otherwise Caddy aborts the TLS handshake) and
    **No TLS Verify** on.

## Setup (fresh host)

1. Install the three binaries into `~/bin`: `caddy` (with Cloudflare DNS plugin),
   `oauth2-proxy`, and `cloudflared` (arm64 — rp2 is a Pi 4):
   ```bash
   curl -fsSL -o ~/bin/cloudflared \
     https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64
   chmod +x ~/bin/cloudflared
   ```
2. Install the units from [`systemd/`](systemd/) into `/etc/systemd/system/`,
   then `sudo systemctl daemon-reload && sudo systemctl enable --now gatekeeper-{oauth,caddy,tunnel}`.
3. **DNS / Google OAuth / Tunnel:** A records (or tunnel CNAMEs) in Cloudflare;
   add `https://auth.carlolm.com/oauth2/callback` to the Google OAuth client's
   Authorized Redirect URIs; create the `.env` secrets below.

## Operations

```bash
# All on rp2.
sudo systemctl restart gatekeeper-oauth     # after editing allowed_emails.txt / oauth2-proxy.cfg
sudo systemctl restart gatekeeper-caddy      # after editing Caddyfile
systemctl status gatekeeper-tunnel           # tunnel health
journalctl -u gatekeeper-caddy -f            # live logs (swap unit name as needed)

# Update who can log in: edit allowed_emails.txt → commit/push → pull on rp2 →
sudo systemctl restart gatekeeper-oauth
```

## Secrets

`.env` is gitignored and lives only on rp2. Keys:

- `CLOUDFLARE_API_TOKEN` — Caddy's DNS-01 challenge
- `TUNNEL_TOKEN` — the `rp2-apps` Cloudflare Tunnel

`oauth2-proxy.cfg` (also gitignored) holds the Google client id/secret, cookie
secret, and points at `allowed_emails.txt`.

Reference values:
- **Redirect URI:** `https://auth.carlolm.com/oauth2/callback`
- **Allowed emails:** see `allowed_emails.txt` (source of truth)

> **Security TODO:** `gatekeeper-caddy.service` on rp2 currently has the
> `CLOUDFLARE_API_TOKEN` *inline* (`Environment=`). The repo copy uses
> `EnvironmentFile=.env` instead — migrate the live unit to match when rotating
> the token.

## Roadmap

- ~~Expose `nutrition` externally via Cloudflare Tunnel, Google-gated.~~ ✅ done.
- Home Assistant external exposure (parallel task) lives in the **`carlo-home`**
  repo on rp1, not here.
