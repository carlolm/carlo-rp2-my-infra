# carlo-rp2-my Internal Infrastructure

Canonical config **and** runbook for the internal web apps on `carlo-rp2-my`,
fronted by Caddy and gated behind Google OAuth2. **This repo is the source of
truth** — deploy by pushing here and pulling on rp2; never edit in place.

## Host

- **Machine:** `carlo-rp2-my` — Tailscale IP `100.111.137.61`
- **Repo on rp2:** `/home/carlo/carlo-rp2-my-infra`
- **Local working dir (Mac):** `~/dev/carlo-rp2-my-infra`
- **Deploy flow:** edit on Mac → commit → push → `git pull` on rp2 → `docker compose up -d`

## Apps & ports

All subdomains resolve to rp2's Tailscale IP and are **Tailscale-only today**
(they time out from the public internet). Upstreams are local services on rp2.
Ports are authoritative from the `Caddyfile`.

| App | URL | Upstream |
|---|---|---|
| Gatekeeper (OAuth2) | https://auth.carlolm.com | `127.0.0.1:4180` |
| Infra Dashboard | https://infra.carlolm.com | `127.0.0.1:7100` |
| Nutrition (frontend) | https://nutrition.carlolm.com | `127.0.0.1:7200` |
| Nutrition (API) | `nutrition.carlolm.com/api/*` + https://api-nutrition.carlolm.com | `127.0.0.1:7201` |
| Newsletters | https://news.carlolm.com | `127.0.0.1:7300` |

## Architecture

- **Caddy** (`slothcroissant/caddy-cloudflaredns`) — reverse proxy + automatic
  HTTPS via the Cloudflare **DNS-01** challenge (no public ports needed for
  certs). Each gated site imports the `(gated)` snippet, which `forward_auth`s
  to oauth2-proxy and redirects unauthenticated requests to the login endpoint.
- **OAuth2 Proxy** — the "Gatekeeper." Forces Google login; only addresses in
  `allowed_emails.txt` pass. Cookie domain `.carlolm.com`, so one login covers
  every subdomain.
- **Network:** docker network `internal_apps_network`.

## Setup

1. **DNS:** A records in Cloudflare for each subdomain → rp2's Tailscale IP.
2. **Google OAuth:** in Google Cloud Console, add
   `https://auth.carlolm.com/oauth2/callback` to Authorized Redirect URIs.
3. **Environment:** create `.env` (gitignored) with the secrets below.
4. **Deploy:** `docker compose up -d`

## Operations

```bash
# All commands run on rp2, in ~/carlo-rp2-my-infra

# Apply config changes / (re)deploy
docker compose up -d

# Live logs
docker compose logs -f caddy
docker compose logs -f oauth2-proxy

# Update who can log in
#   1. edit allowed_emails.txt  2. commit + push  3. pull on rp2
docker compose up -d   # picks up the new file
```

> **Reconcile (one-time):** an older deployment ran Caddy + oauth2-proxy as
> systemd units (`gatekeeper-caddy.service`, `gatekeeper-oauth.service`) from
> `~/bin/`. The Docker Compose stack in this repo supersedes that. Verify the
> legacy units are stopped/disabled on rp2 so they don't fight the containers
> for ports 80/443/4180.

## Secrets

`.env` is gitignored and lives only on rp2. Required keys:

- `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` — OAuth client
- `OAUTH2_PROXY_COOKIE_SECRET` — cookie signing key
- `CLOUDFLARE_API_TOKEN` — for Caddy's DNS-01 challenge
- `TUNNEL_TOKEN` — token for the remotely-managed `rp2-apps` Cloudflare Tunnel (cloudflared)

Reference values:
- **Redirect URI:** `https://auth.carlolm.com/oauth2/callback`
- **Allowed emails:** see `allowed_emails.txt` (source of truth)

## Roadmap

- **Expose `nutrition` externally** via a Cloudflare Tunnel, still gated by
  Google auth (today it's Tailscale-only). Design modeled on the rp1 wife's-site
  tunnel. Decide: reuse the existing oauth2-proxy gate, or move the gate to
  Cloudflare Zero Trust Access.

> Note: Home Assistant external exposure (the parallel task) lives in the
> **`carlo-home`** repo on rp1, not here.
