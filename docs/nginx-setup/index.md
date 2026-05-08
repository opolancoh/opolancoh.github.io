# Nginx Reverse Proxy Setup

Guide for setting up nginx as a reverse proxy on a Debian-based Linux system (Raspberry Pi, Ubuntu, Debian, etc.). The examples below use a home server behind a router with Cloudflare managing DNS, but the nginx configuration applies to any environment.

## Contents

- [Context](#context)
- [Infrastructure Overview](#infrastructure-overview)
- [Installation](#installation)
- [Sites](#sites)
  - [`api.spenbify.com`](#apispenbifycom)
  - [`ikobit.com`](#ikobitcom)
- [Issues & Fixes](#issues--fixes)
- [Notes](#notes)

## Context

Cloudflare manages DNS for the domains and proxies traffic to the public IP (`38.188.254.30`). The router NATs inbound traffic on port 80 to a home server at `192.168.58.200`, where nginx terminates the connection and proxies to individual apps running on internal ports.

## Infrastructure Overview

```
Internet → Cloudflare (DNS + Proxy) → Public IP 38.188.254.30 → Router NAT → 192.168.58.200:80 → nginx → App
```

### Port Assignments

| Domain | App | Internal Port |
|---|---|---|
| `api.spenbify.com` | .NET API | 7001 |
| `ikobit.com` / `www.ikobit.com` | Astro (static, Docker) | 7010 |

---

## Installation

```bash
sudo apt update
sudo apt install nginx -y
```

### Verify nginx is running

```bash
sudo systemctl status nginx --no-pager
```

The package starts nginx automatically and ships a default site (symlinked at `/etc/nginx/sites-enabled/default`) that listens on port 80 and serves a "Welcome to nginx!" page.

### Remove the default site

Remove it so your own configs (e.g. `api.spenbify.com`) take over port 80 cleanly:

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl reload nginx

# confirm sites-enabled is empty (your configs go here later)
ls -l /etc/nginx/sites-enabled/
```

Expected output (an empty directory):

```
total 0
```

### Confirm nginx is listening on port 80

```bash
sudo ss -tlnp | grep nginx
```

You should see nginx bound to `0.0.0.0:80` (and `[::]:80` for IPv6). If nothing prints, nginx isn't listening — check the service status above.

---

## Sites

Each site is set up in four steps: create a config file under `/etc/nginx/sites-available/`, enable it once by symlinking into `sites-enabled/`, apply the config (test + reload nginx), then test it from another machine on the LAN. The `-H "Host: ..."` header is required during testing because nginx routes requests based on the `server_name`, not just the IP.

> **Note:** the symlink (`ln -s`) is a one-time action when adding a new site. Whenever you later edit an existing config, only the **Apply the config** step (`nginx -t && systemctl reload nginx`) needs to be re-run.

### `api.spenbify.com`

**1. Create the config**

```bash
sudo nano /etc/nginx/sites-available/api.spenbify.com
```

```nginx
server {
    listen 80;
    server_name api.spenbify.com;

    location / {
        proxy_pass http://localhost:7001;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**2. Enable the site** (one-time)

```bash
sudo ln -s /etc/nginx/sites-available/api.spenbify.com /etc/nginx/sites-enabled/
```

**3. Apply the config** (also re-run after any future edit)

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**4. Test**

```bash
curl -i -H "Host: api.spenbify.com" http://192.168.58.200/health
```

Expect a `HTTP/1.1 200 OK` status line followed by whatever the upstream API returns for `/health` (e.g. `Healthy` for a default ASP.NET health check). Any non-200 means the request reached nginx but the upstream isn't responding as expected — check the app on `localhost:7001` directly.

---

### `ikobit.com`

**1. Create the config**

```bash
sudo nano /etc/nginx/sites-available/ikobit.com
```

```nginx
server {
    listen 80;
    server_name ikobit.com www.ikobit.com;

    # Prevent nginx from appending the internal port in redirects
    port_in_redirect off;

    location / {
        proxy_pass http://localhost:7010;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**2. Enable the site** (one-time)

```bash
sudo ln -s /etc/nginx/sites-available/ikobit.com /etc/nginx/sites-enabled/
```

**3. Apply the config** (also re-run after any future edit)

```bash
sudo nginx -t
sudo systemctl reload nginx
```

**4. Test**

```bash
curl -i -H "Host: ikobit.com" http://192.168.58.200/
```

Expect a `HTTP/1.1 200 OK` status line and an HTML body starting with `<!DOCTYPE html>`. If you get a 502, the Astro container isn't reachable on `localhost:7010`.

---

## Issues & Fixes

### Astro internal links breaking behind proxy
**Symptom:** Page links worked on `192.168.58.200:7010` directly but broke when accessed through nginx on port 80 — nginx was appending the internal port (`:7010`) to redirect URLs.

**Fix:** Add `port_in_redirect off;` to the nginx server block. This prevents nginx from including the upstream port in `Location` headers when issuing redirects.

---

## Notes

- Cloudflare handles SSL termination — nginx communicates with Cloudflare over HTTP on port 80, which is fine since that leg is within Cloudflare's edge network.
- The Astro app runs as a Docker container. The .NET API runs directly on the server.
- Both `spenbify.com` and `ikobit.com` resolve to the same public IP (`38.188.254.30`). nginx differentiates them via the `server_name` directive.
