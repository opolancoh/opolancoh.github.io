# Cloudflare Tunnel Setup

Expose apps on a Raspberry Pi 4 to the internet using Cloudflare Tunnel (`cloudflared`), alongside an existing nginx reverse proxy. A single tunnel handles multiple hostnames across multiple Cloudflare zones; nginx routes each request by `Host` header.

## Contents

- [Context](#context)
- [Infrastructure Overview](#infrastructure-overview)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Create the Tunnel](#create-the-tunnel)
- [Run as a systemd Service](#run-as-a-systemd-service)
- [Zones](#zones)
  - [`domain-a.com` zone — Worker fallback pattern](#domain-acom-zone--worker-fallback-pattern)
  - [`domain-b.com` zone — Direct tunnel pattern](#domain-bcom-zone--direct-tunnel-pattern)
- [Notes](#notes)

## Context

`cloudflared` opens an outbound connection from the Pi to Cloudflare's edge — no inbound ports on the router. Each zone picks one of two patterns:

| Pattern | When to use |
|---|---|
| **Direct tunnel** | Tunnel owns the public hostname end-to-end. Simple. No fallback. |
| **Worker fallback** | A Cloudflare Worker owns the public hostname and routes to either the tunnel or a cloud backend. One extra DNS hostname; gains a manual failover switch. |

Different zones can use different patterns at the same time. Below: `domain-a.com` uses Worker fallback (so the API can fail over to a cloud backend), and `domain-b.com` uses direct tunnel.

## Infrastructure Overview

**Direct tunnel (`domain-b.com`):**

```
Internet → Cloudflare Edge → Tunnel → nginx:80 → Static site:7010
```

**Worker fallback (`api.domain-a.com`):**

```
Internet → Cloudflare Edge → Worker (api.domain-a.com)
                                ├── PRIMARY:  api-pi.domain-a.com → Tunnel → nginx:80 → API:7001
                                └── FALLBACK: <cloud-backend>
```

| Domain | App | Internal Port |
|---|---|---|
| `api.domain-a.com` | API | 7001 |
| `domain-b.com` / `www.domain-b.com` | Static site (Docker) | 7010 |

`cloudflared` only talks to nginx on port 80; nginx routes from there to the apps.

---

## Prerequisites

- Cloudflare account managing the domains
- nginx configured with virtual hosts for each domain (see [Nginx Reverse Proxy Setup](../../nginx-setup/index.md))
- Raspberry Pi 4 running 64-bit Raspberry Pi OS (arm64)

---

## Installation

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb -o cloudflared.deb
sudo dpkg -i cloudflared.deb
rm cloudflared.deb
cloudflared --version
```

---

## Create the Tunnel

These steps run once.

### Authenticate

This step is **only** needed for the `cloudflared tunnel route dns ...` CLI shortcut. If you create DNS records manually in the dashboard (shown per zone below), you can skip authentication entirely — the tunnel itself works without it.

If you do use the CLI, **run `cloudflared tunnel login` once per zone you want to manage**. The `cert.pem` issued by the command is scoped to the **one** zone selected during the browser flow; each additional zone requires another login round.

```bash
cloudflared tunnel login
```

Since the Pi is headless, `cloudflared` prints a URL. Open it in a browser, log in, and select a zone (e.g. `domain-a.com`). The Pi terminal then unblocks and writes `~/.cloudflared/cert.pem`. **Re-run the command and select each additional zone** (e.g. `domain-b.com`) before continuing.

> **Common mistake:** skipping the per-zone re-run. If `cert.pem` doesn't cover a zone, `cloudflared tunnel route dns ... <hostname-in-that-zone>` won't fail — it silently creates a wrong-zone record like `domain-b.com.domain-a.com`, the real zone has no record at all, and the hostname returns `HTTP 522` in production with no obvious error to point at.

### Create the tunnel

```bash
cloudflared tunnel create <tunnel-name>
```

Use a name that identifies the machine (e.g. `rpi-4-v1`). Output:

```
Tunnel credentials written to /home/<your-username>/.cloudflared/<tunnel-id>.json.
Created tunnel <tunnel-name> with id <tunnel-id>
```

Retrieve the `<tunnel-id>` (UUID) any time with `cloudflared tunnel list`.

### Create the base config file

```bash
sudo nano /etc/cloudflared/config.yml
```

```yaml
tunnel: <tunnel-name>
credentials-file: /home/<your-username>/.cloudflared/<tunnel-id>.json

ingress:
  # Per-zone ingress rules go here, before the catch-all.

  - service: http_status:404
```

---

## Run as a systemd Service

```bash
sudo cloudflared --config /etc/cloudflared/config.yml service install
sudo systemctl enable --now cloudflared
sudo systemctl status cloudflared --no-pager
```

The `--config` flag is required because under `sudo`, `~` resolves to `/root` and the default config lookup fails.

Confirm the tunnel registered with Cloudflare's edge:

```bash
sudo journalctl -u cloudflared -n 50 --no-pager
```

Look for four `Registered tunnel connection connIndex=0..3` lines (Cloudflare opens four redundant connections by default).

---

## Zones

Each zone follows the same flow inside its own subsection:

1. **Add ingress entries** to `/etc/cloudflared/config.yml`, above the catch-all.
2. **Apply the config** — validate, then restart `cloudflared`. Re-run after any later edit.
3. **Create DNS records** in the zone — Worker route, tunnel CNAMEs, or both.
4. **Test**.

DNS records use the **manual dashboard path** because it works regardless of which zones your `cert.pem` covers. The CLI shortcut (`cloudflared tunnel route dns ...`) is mentioned where it applies.

### `domain-a.com` zone — Worker fallback pattern

A Worker owns `api.domain-a.com` and routes to the tunnel (via `api-pi.domain-a.com`) or a cloud backend. `httpHostHeader` rewrites the `Host` header so nginx's `server_name api.domain-a.com` still matches.

**1. Add ingress entry**

```yaml
- hostname: api-pi.domain-a.com
  service: http://localhost:80
  originRequest:
    httpHostHeader: api.domain-a.com
```

**2. Apply the config**

```bash
cloudflared tunnel ingress validate
sudo systemctl restart cloudflared
```

**3. Create DNS records**

In the **`domain-a.com` zone** → DNS → Records:

*`api-pi.domain-a.com` → tunnel:*
- **Type:** `CNAME`, **Name:** `api-pi`, **Target:** `<tunnel-id>.cfargotunnel.com`, **Proxied**

*`api.domain-a.com` → Worker:* created automatically when you assign the Worker route in step 4. If a record already exists, delete it first.

> CLI alternative for the `api-pi` record (only if your cert covers `domain-a.com`): `cloudflared tunnel route dns <tunnel-name> api-pi.domain-a.com`

**4. Deploy the Cloudflare Worker**

In the dashboard, create a Worker and assign it to the route `api.domain-a.com/*`.

```js
// API Proxy - forwards api.domain-a.com to the active backend.
// To fail over, swap ACTIVE_BACKEND to FALLBACK_BACKEND and redeploy.

const PRIMARY_BACKEND  = "api-pi.domain-a.com";       // Tunnel → Pi
const FALLBACK_BACKEND = "your-api.cloud-provider.com";

const ACTIVE_BACKEND = PRIMARY_BACKEND;
const BACKEND_SCHEME = "https";

export default {
  async fetch(request) {
    const url = new URL(request.url);
    url.hostname = ACTIVE_BACKEND.split(":")[0];
    url.protocol = BACKEND_SCHEME + ":";
    url.port = ACTIVE_BACKEND.includes(":") ? ACTIVE_BACKEND.split(":")[1] : "";

    const proxiedRequest = new Request(url, request);
    proxiedRequest.headers.set("Host", url.hostname);
    return fetch(proxiedRequest);
  }
};
```

**5. Test**

```bash
curl -i https://api-pi.domain-a.com/health   # tunnel directly
curl -i https://api.domain-a.com/health      # via Worker
```

Both should return `HTTP/2 200`. If `api-pi` works but `api` doesn't, the issue is in the Worker (check Worker logs in the dashboard).

---

### `domain-b.com` zone — Direct tunnel pattern

The tunnel owns the hostnames end-to-end. No Worker.

**1. Add ingress entries**

```yaml
- hostname: domain-b.com
  service: http://localhost:80
- hostname: www.domain-b.com
  service: http://localhost:80
```

**2. Apply the config**

```bash
cloudflared tunnel ingress validate
sudo systemctl restart cloudflared
```

**3. Create DNS records**

In the **`domain-b.com` zone** → DNS → Records. If the apex already has an A-record from a previous (non-tunnel) setup, **delete it first** — leaving it in place routes traffic to the old origin and produces a 522.

*Apex `domain-b.com` → tunnel:*
- **Type:** `CNAME`, **Name:** `@`, **Target:** `<tunnel-id>.cfargotunnel.com`, **Proxied**

*`www.domain-b.com` → apex:*
- **Type:** `CNAME`, **Name:** `www`, **Target:** `domain-b.com`, **Proxied**

CNAMEs at the apex are allowed; Cloudflare uses CNAME flattening transparently. Chaining `www → apex → tunnel` means future tunnel changes only require updating the apex record.

> CLI alternative (only if your cert covers `domain-b.com`): `cloudflared tunnel route dns <tunnel-name> domain-b.com` and `... www.domain-b.com`. **Caveat:** if the cert doesn't cover the zone, the CLI silently creates wrong-zone records like `domain-b.com.domain-a.com` instead of failing — use the dashboard to avoid this.

**4. Test**

```bash
curl -i https://domain-b.com/
curl -i https://www.domain-b.com/
```

Both should return `HTTP/2 200`.

To isolate the tunnel from the Pi side, test nginx directly from the LAN:

```bash
curl -i -H "Host: domain-b.com" http://<pi-lan-ip>/
```

If LAN works but the public URL doesn't, the problem is between Cloudflare and the tunnel — not nginx.

---

## Notes

- Cloudflare Tunnel is free with any Cloudflare account.
- A single tunnel can serve hostnames across multiple zones, as shown here.
- Traffic between the Pi and Cloudflare's edge is encrypted by default. The HTTP connection between `cloudflared` and nginx is local to the machine.
- The `cloudflared` package manages its own updates automatically.
