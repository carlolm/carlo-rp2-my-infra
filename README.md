# carlo-rp2-my Internal Infrastructure

This repository manages the internal web applications on `carlo-rp2-my` behind Google OAuth2 authentication.

## Apps Protected
- **Infra Dashboard**: https://infra.carlolm.com
- **Nutrition Site**: https://nutrition.carlolm.com
- **Newsletters**: https://news.carlolm.com

## Architecture
- **Caddy**: Reverse proxy handling subdomains and SSL (Internal DNS-01 challenge).
- **OAuth2 Proxy**: The "Gatekeeper" that forces Google Login.
- **Tailscale**: These domains point to the Tailscale IP of `rp2`.

## Setup
1. **DNS**: Create A records in Cloudflare for the subdomains above pointing to your `rp2` Tailscale IP.
2. **Google OAuth**: 
   - Go to Google Cloud Console.
   - Add `https://auth.carlolm.com/oauth2/callback` to Authorized Redirect URIs.
3. **Environment**: Update `.env` with your Google Client ID, Secret, and Cloudflare Token.
4. **Deploy**:
   ```bash
   docker compose up -d
   ```

## Note on HTTPS
Caddy uses the Cloudflare DNS challenge to get SSL certificates without opening ports 80/443 to the public internet. This ensures you get a green lock even on your internal network.
