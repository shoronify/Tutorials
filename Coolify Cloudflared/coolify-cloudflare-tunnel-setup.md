# Coolify + Cloudflare Tunnel Setup — cloudmonk.net

This sets up two things at once:
1. `coolify.cloudmonk.net` → your Coolify dashboard (with realtime + terminal working over the tunnel)
2. A wildcard + apex route so **every future app you deploy in Coolify just works** — no more trips back to the Cloudflare dashboard.

**Assumption:** `cloudflared` is running as a system service directly on the VPS (not inside a Docker container), since you said you already installed and connected it. If it's actually running inside Docker without `--network host`, replace `localhost` below with `host.docker.internal` (and add `--add-host=host.docker.internal:host-gateway` to how it's run).

---

## 1. Install Coolify (if not already done)

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

Leave the defaults: dashboard on port `8000`, proxy (Traefik) on `80`/`443`, realtime (Soketi) on `6001`, terminal websocket on `6002`. Don't try to remap these — the routes below are built around the defaults.

---

## 2. Configure Cloudflare Tunnel routes (Public Hostnames)

Go to **Cloudflare Zero Trust dashboard → Networking → Tunnels → your tunnel → Configure → Routes → Add route → Published Application**.

Add these **in this exact order** (order matters — Cloudflare matches top to bottom, so specific rules must sit above the wildcard):

| # | Subdomain | Domain | Path | Service URL | Purpose |
|---|---|---|---|---|---|
| 1 | `coolify` | cloudmonk.net | `/terminal/ws` | `http://localhost:6002` | Coolify's in-browser terminal |
| 2 | `coolify-realtime` | cloudmonk.net | *(empty)* | `http://localhost:6001` | Realtime/websocket server (Soketi) |
| 3 | `coolify` | cloudmonk.net | *(empty)* | `http://localhost:8000` | Coolify dashboard itself |
| 4 | *(empty/@)* | cloudmonk.net | *(empty)* | `http://localhost:80` | Apex domain, if you ever deploy an app on bare `cloudmonk.net` |
| 5 | `*` | cloudmonk.net | *(empty)* | `http://localhost:80` | **Catch-all** — routes every subdomain to Coolify's proxy (Traefik) |

Row 5 is the important one for your "don't touch Cloudflare again" requirement: it hands off routing to Coolify's own reverse proxy, which reads the `Host` header and forwards to whichever app you've assigned that domain to inside Coolify. So `myapp.cloudmonk.net`, `blog.cloudmonk.net`, etc. all resolve automatically the moment you type them into Coolify's app settings and deploy — no new Cloudflare route needed.

**Caveat:** Cloudflare's free wildcard only covers *one* subdomain level. `*.cloudmonk.net` covers `app.cloudmonk.net` but not `staging.app.cloudmonk.net`. If you need nested subdomains, add a specific route for that one, or get Cloudflare's Advanced Certificate Manager.

Cloudflare auto-creates the CNAME DNS records (proxied, orange-cloud) when you add these routes. If a record already exists for a hostname it won't overwrite it — if something doesn't resolve, check **DNS → Records** and manually point it to `<tunnel-id>.cfargotunnel.com` (proxied).

---

## 3. Point Coolify at its realtime server through the tunnel

SSH in and edit the env file:

```bash
nano /data/coolify/source/.env
```

Add these two lines (don't touch the existing `APP_ID`, `PUSHER_APP_*`, `REDIS_PASSWORD`, etc. — those already exist from install):

```
PUSHER_HOST=coolify-realtime.cloudmonk.net
PUSHER_PORT=443
```

Apply the change:

```bash
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

---

## 4. Set the instance domain inside Coolify

In the Coolify dashboard: **Settings → General/Instance settings** (also whatever "Domains" field appears when you deploy an app):

- Set it as `http://coolify.cloudmonk.net` — **http, not https**.
- Same rule for every app you deploy later: enter the domain as `http://yourapp.cloudmonk.net` in Coolify.

Reason: Cloudflare terminates TLS at the edge; Coolify's Traefik only ever sees plain HTTP over the tunnel. If you configure `https://` on the Coolify side without also installing a real cert on Traefik, you'll get a `TOO_MANY_REDIRECTS` loop. (There's an advanced "Full HTTPS/TLS" path using a Cloudflare Origin Certificate on Traefik if you later need end-to-end HTTPS for something like JWT/OAuth callbacks — happy to walk through that if you hit that need.)

---

## 5. Cloudflare dashboard security settings

| Setting | Where | Value |
|---|---|---|
| Encryption mode | SSL/TLS → Overview | **Full (Strict)** |
| Always Use HTTPS | SSL/TLS → Edge Certificates | **On** |
| Minimum TLS version | SSL/TLS → Edge Certificates | **TLS 1.2** (enable 1.3 too) |
| Automatic HTTPS Rewrites | SSL/TLS → Edge Certificates | On (avoids mixed-content issues) |
| Bot Fight Mode | Security → Bots | **On** (free tier) — cuts down credential-stuffing hits on your login page |
| WAF managed ruleset | Security → WAF | Enable the free Cloudflare Managed Ruleset |
| Cloudflare 2FA | My Profile → Authentication | **On**, on your Cloudflare account itself |

### Strongly recommended: gate the dashboard with Cloudflare Access
Since `coolify.cloudmonk.net` is now internet-reachable, put Cloudflare Zero Trust Access in front of it so visitors need to pass an email OTP (or SSO) *before* they even see Coolify's login page:

**Zero Trust → Access → Applications → Add an application → Self-hosted**
- Application domain: `coolify.cloudmonk.net`
- Policy: allow only your email (or your identity provider), everyone else blocked
- Repeat for `coolify-realtime.cloudmonk.net` so the websocket hostname is covered too (skip this one if it causes realtime connection issues — Access uses a cookie that should carry over fine within the same browser session, but test it)

Also turn on 2FA inside Coolify itself (**Settings → Two-Factor Authentication**) as a second layer.

---

## 6. Lock down the VPS firewall

This is the actual security win of using a Tunnel: `cloudflared` only makes **outbound** connections to Cloudflare — nothing needs to be open for inbound traffic at all. Close everything except SSH:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH          # or your custom SSH port, e.g. sudo ufw allow 2222/tcp
sudo ufw enable
sudo ufw status verbose
```

You do **not** need to open 80, 443, 8000, 6001, or 6002 to the internet — `cloudflared` reaches them over `localhost` locally, and the default-deny policy blocks any direct external hit on your VPS's public IP. Double check your VPS provider's own network/security-group firewall (Hetzner Cloud Firewall, DigitalOcean Firewall, etc.) is equally locked down if you use one alongside `ufw`.

---

## 7. Verify

1. Visit `https://coolify.cloudmonk.net` — dashboard should load over HTTPS.
2. Visit `https://coolify.cloudmonk.net/realtime` — you should see a test event notification confirming the websocket/Soketi connection works.
3. Open a deployed app's **Terminal** tab in Coolify and confirm it connects (tests port 6002).
4. Deploy a throwaway test app, set its domain to `http://test.cloudmonk.net` in Coolify, hit deploy, and confirm it's reachable — no Cloudflare changes required, proving the wildcard route works.

---

## From now on, when deploying new apps

Just add the domain (`http://whatever.cloudmonk.net` or `http://cloudmonk.net` for apex) in the app's **Domains** field in Coolify and deploy. The wildcard tunnel route + Coolify's Traefik proxy handle the rest automatically.
