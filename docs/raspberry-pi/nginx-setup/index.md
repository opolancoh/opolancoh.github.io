# Raspberry Pi Nginx Reverse Proxy Setup

## Context

A Raspberry Pi at home (`192.168.58.200`) serves as a local server hosting multiple apps. The router maps inbound internet traffic to the Pi, and Cloudflare manages DNS for the domains. All traffic flows through nginx on port 80, which proxies to the individual apps running on internal ports.

## Infrastructure Overview

```
Internet → Cloudflare (DNS + Proxy) → Public IP 38.188.254.30 → Router NAT → Pi:80 → nginx → App
```

### Port Assignments

| Domain | App | Internal Port |
|---|---|---|
| `api-qa.spenbify.com` | .NET API | 7001 |
| `ikobit.com` / `www.ikobit.com` | Astro (static, Docker) | 7010 |

---

## Installation

```bash
sudo apt update
sudo apt install nginx -y
sudo rm /etc/nginx/sites-enabled/default
```

---

## nginx Configs

### `/etc/nginx/sites-available/api-qa.spenbify.com`

```nginx
server {
    listen 80;
    server_name api-qa.spenbify.com;

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

### `/etc/nginx/sites-available/ikobit.com`

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

---

## Enabling Sites

```bash
sudo ln -s /etc/nginx/sites-available/api-qa.spenbify.com /etc/nginx/sites-enabled/
sudo ln -s /etc/nginx/sites-available/ikobit.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## Testing

### Check nginx is listening on port 80
```bash
sudo ss -tlnp | grep nginx
```

### Test locally from another machine on the network
```bash
curl -H "Host: api-qa.spenbify.com" http://192.168.58.200/health
curl -H "Host: ikobit.com" http://192.168.58.200
```

The `-H "Host: ..."` header is required because nginx routes requests based on the `server_name`, not just the IP.

---

## Issues & Fixes

### Astro internal links breaking behind proxy
**Symptom:** Page links worked on `192.168.58.200:7010` directly but broke when accessed through nginx on port 80 — nginx was appending the internal port (`:7010`) to redirect URLs.

**Fix:** Add `port_in_redirect off;` to the nginx server block. This prevents nginx from including the upstream port in `Location` headers when issuing redirects.

---

## Notes

- Cloudflare handles SSL termination — nginx communicates with Cloudflare over HTTP on port 80, which is fine since that leg is within Cloudflare's edge network.
- The Astro app runs as a Docker container. The .NET API runs directly on the Pi.
- Both `spenbify.com` and `ikobit.com` resolve to the same public IP (`38.188.254.30`). nginx differentiates them via the `server_name` directive.
