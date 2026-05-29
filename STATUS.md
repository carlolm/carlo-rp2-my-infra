# carlo-rp2-my — Status & Handover

**Last updated:** 2026-05-29

Checkpoint for the external-exposure + cleanup effort. Read alongside
[`README.md`](README.md) (the how-it-runs reference). Home Assistant work is
tracked separately in the `carlo-home` repo (rp1).

## Done (2026-05-29)

- **`nutrition.carlolm.com` is PUBLIC + Google-gated** via the `rp2-apps`
  Cloudflare Tunnel.
  - cloudflared installed on rp2 at `~/bin/cloudflared`, run by
    `gatekeeper-tunnel.service` (enabled, auto-starts on boot).
  - Tunnel public routes: `nutrition.carlolm.com` and `auth.carlolm.com` →
    `https://localhost:443`, each with **Origin Server Name = its hostname**
    (required SNI) + **No TLS Verify ON**.
  - Verified from the public internet: `GET /` → 302 to Google login;
    `GET /api/today` → 302 (the API is gated too).
  - DNS: `nutrition` + `auth` are now proxied tunnel CNAMEs; `infra`, `news`,
    `api-nutrition` remain `A → 100.111.137.61` (Tailscale-only).
- **Allowed emails** now include `michellekphua@gmail.com`,
  `kaigee.ism@gmail.com` (live; `gatekeeper-oauth` restarted).
- **Repo corrected** to the native-systemd reality; `docker-compose.yml` removed;
  `systemd/` reference units added. (commit `404c38b`)

## Verified secure

- API (7201) + frontend (7200) bind to `127.0.0.1` only — reachable only via Caddy.
- Frontend (Next.js) calls the API **same-origin `/api/*`** in the browser, and
  `127.0.0.1:7201` for SSR. No absolute URL baked into the client build.
  `api-nutrition.carlolm.com` is **not** used by the app.
- `/api` is gated; `api-nutrition.carlolm.com` is also gated **and** Tailscale-only.

## Open follow-ups (resume here)

1. **SECURITY — rotate `CLOUDFLARE_API_TOKEN`** (it surfaced in cleartext in
   `gatekeeper-caddy.service` and in a session log).
   - New token in CF dashboard (Zone→DNS→Edit on `carlolm.com`).
   - rp2: put it in `~/carlo-rp2-my-infra/.env`, then swap the live unit to the
     repo's EnvironmentFile version: `sudo cp systemd/gatekeeper-caddy.service
     /etc/systemd/system/ && sudo systemctl daemon-reload && sudo systemctl
     restart gatekeeper-caddy`; confirm certs in `journalctl -u gatekeeper-caddy`.
   - Also update wherever it's used on `sanjose-alpha-rancher`.
2. **HA external exposure (rp1)** — tracked in `carlo-home` repo. Plan: a
   **dedicated `rp1-home` tunnel** (2nd cloudflared in the carlo-home compose) +
   **Cloudflare Access** (Google gate); expose a NEW hostname (e.g.
   `home-ext.carlolm.com`) for **browser only**; keep the iPhone Companion app on
   `home-remote.carlolm.com` (Tailscale) so Access doesn't break it.
   **Blocked on:** create the `rp1-home` tunnel + token, then wire cloudflared
   into `carlo-home/docker-compose.yml`.
3. **rp2 NVMe (RTL9210 `0bda:9210`) instability is HARDWARE.** Software quirks
   already applied (`usb-storage.quirks=…:u` → UAS off; `usbcore.autosuspend=-1`)
   and a powered hub didn't help. Options: better-chipset enclosure
   (ASMedia/JMicron), or Pi 5 + NVMe HAT (native). SD-only can't hold the ~313GB.
   **Verify the 138GB of photos are backed up off rp2** (sole copy on a flaky drive).
4. *(Optional)* expose `news` / `infra` publicly later — same two steps as
   nutrition (remove the A record, add a tunnel route with Origin Server Name).

## Key facts

- **rp2 = native systemd, no Docker.** rp1 = Docker (HA, caddy, michelle-site, cloudflared).
- Restart after config change: `sudo systemctl restart gatekeeper-oauth gatekeeper-caddy`
  (tunnel unit is `gatekeeper-tunnel`).
- rp2 Tailscale IP `100.111.137.61`; repo on rp2 `/home/carlo/carlo-rp2-my-infra`.
