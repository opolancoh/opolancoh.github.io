# Manual Failover Strategy with Cloudflare Workers

This document describes a strategy to route traffic from a stable public domain (e.g. `api.spenbify.com`) to different backend services, with the ability to manually switch between them when the primary service goes down.

## Context

- The mobile app must **always** point to a single, stable domain (`api.spenbify.com`).
- The primary backend is a self-hosted server (e.g. a Raspberry Pi at home with a static public IP).
- A fallback backend runs on Google Cloud Run for the times when the primary is offline.
- Switching between backends must not require any changes in the mobile app.
- Failover is manual (not automatic), because switching backends may require manual steps like database synchronization. A short downtime window during the switch is acceptable.

## The Problem

A common first attempt is to point the domain directly at Cloud Run using a proxied CNAME in Cloudflare:

```
api.spenbify.com  →  CNAME  →  xxxxx.run.app  (proxied)
```

This fails with a **404 from Google**. The reason is that Cloud Run routes incoming requests based on the `Host` header. When Cloudflare forwards the request, the `Host` header is `api.spenbify.com`, which Cloud Run does not recognize as a mapped service, so it returns a generic 404.

Cloud Run's native "Domain Mappings" feature can solve this, but:

- It is still in preview in many regions.
- It is not available in all regions (for example, `us-east5` does not support it at the time of writing).
- The global variant is also in preview and adds operational complexity.

## The Solution

Use a **Cloudflare Worker** as a lightweight reverse proxy in front of the domain. The Worker:

1. Intercepts every request to `api.spenbify.com`.
2. Rewrites the URL's hostname, scheme, and port to point to the currently active backend.
3. Rewrites the `Host` header to match the backend's hostname (so Cloud Run accepts the request).
4. Forwards the request and returns the response to the client.

Switching backends is as simple as editing a single line in the Worker code and clicking **Deploy**. The change takes effect globally within seconds. The mobile app is never aware of the change.

### Trade-offs

**Pros**
- No changes needed in GCP (no domain mappings, no load balancer, no serverless NEG).
- Free on Cloudflare's plan up to 100,000 requests/day.
- Fast to set up and easy to operate.
- Keeps Cloudflare's benefits (WAF, caching, DDoS protection, IP hiding).
- Mobile app configuration never changes.

**Cons**
- Adds a small amount of latency (typically 5–15 ms) per request.
- The backend URLs are embedded in the Worker code; changing a backend requires editing and redeploying the Worker.
- The Cloud Run URL, if discovered, could be accessed directly, bypassing Cloudflare. This can be mitigated by restricting Cloud Run's ingress, but that adds complexity.

## Step-by-Step Setup

### 1. Verify your DNS record in Cloudflare

In Cloudflare, under your domain's **DNS → Records**, confirm or create a record for the subdomain. It can initially point anywhere (it will be replaced when the Worker is attached as a custom domain).

### 2. Create the Worker

1. In the Cloudflare dashboard, go to **Workers & Pages**.
2. Click **Create application → Create Worker** (or **Start with Hello World!**).
3. Give it a descriptive name (e.g. `spenbify-api-proxy`).
4. Click **Deploy** to create it with the default Hello World code.

### 3. Replace the Worker code

1. On the Worker's page, click **Edit code**.
2. Delete the default code and paste the script provided in the [Final Script](#final-script) section below.
3. Update the `PRIMARY_BACKEND` constant with your actual public IP or DDNS hostname.
4. Update the `FALLBACK_BACKEND` constant with your Cloud Run URL (without `https://`).
5. Click **Deploy**.

### 4. Attach the Worker to your custom domain

1. Go back to the Worker's **Overview** page.
2. In the **Domains** panel on the left, click **Add a custom domain**.
3. Enter `api.spenbify.com` (or the subdomain you want to route through the Worker).
4. If Cloudflare complains that the domain is already in use, go to **DNS → Records**, delete the existing record for that subdomain, and try again.
5. Cloudflare will automatically create the DNS record pointing to the Worker.

### 5. Verify

Run a request against the domain:

```bash
curl -v https://api.spenbify.com/
```

You should receive the response from the active backend (not a 404 from Google or a 403 from Cloudflare).

## How to Perform a Failover

When you need to switch between backends:

1. Open the Worker in the Cloudflare dashboard and click **Edit code**.
2. Find the line:
   ```js
   const ACTIVE_BACKEND = FALLBACK_BACKEND;
   ```
3. Change the value to the backend you want to use:
   - To use the primary server: `const ACTIVE_BACKEND = PRIMARY_BACKEND;`
   - To use the fallback: `const ACTIVE_BACKEND = FALLBACK_BACKEND;`
4. Click **Deploy**.
5. The change propagates globally within seconds. No action is required on the mobile app or DNS.

## Final Script

```js
// =============================================================
// Spenbify API Proxy - Cloudflare Worker
// =============================================================
// This Worker receives all traffic for api.spenbify.com and
// forwards it to the currently active backend.
//
// To switch backends, edit the ACTIVE_BACKEND constant below
// and click Deploy.
// =============================================================

// -------------------------------------------------------------
// Available backends
// -------------------------------------------------------------

// Primary API service
// Replace with your static public IP or DDNS hostname.
// Include the port if the server is not on 443 (e.g. "1.2.3.4:8080").
const PRIMARY_BACKEND = "YOUR_PRIMARY_HOST_HERE";

// Fallback API service
const FALLBACK_BACKEND = "spenbify-api-843832646922.us-east5.run.app";

// -------------------------------------------------------------
// Active backend - CHANGE THIS LINE TO PERFORM FAILOVER
// -------------------------------------------------------------
// Use primary:  const ACTIVE_BACKEND = PRIMARY_BACKEND;
// Use fallback: const ACTIVE_BACKEND = FALLBACK_BACKEND;

const ACTIVE_BACKEND = FALLBACK_BACKEND;

// -------------------------------------------------------------
// Backend scheme (http/https)
// -------------------------------------------------------------
// The fallback (Cloud Run) is always HTTPS.
// The primary typically serves HTTP unless it has its own TLS cert.
// Adjust as needed.

const BACKEND_SCHEME = ACTIVE_BACKEND === FALLBACK_BACKEND ? "https" : "http";

// =============================================================
// Proxy logic
// =============================================================

export default {
  async fetch(request) {
    const url = new URL(request.url);

    // Rewrite hostname and scheme to point to the active backend
    url.hostname = ACTIVE_BACKEND.split(":")[0];
    url.protocol = BACKEND_SCHEME + ":";

    // Apply port if the backend includes one (e.g. ":8080")
    if (ACTIVE_BACKEND.includes(":")) {
      url.port = ACTIVE_BACKEND.split(":")[1];
    } else {
      url.port = "";
    }

    // Clone the incoming request with the rewritten URL
    const proxiedRequest = new Request(url, request);

    // Rewrite the Host header to match the backend hostname
    // (required by Cloud Run; harmless for other backends)
    proxiedRequest.headers.set("Host", url.hostname);

    return fetch(proxiedRequest);
  }
};
```

## Notes and Considerations

- **Primary backend port**: if your self-hosted server listens on a non-standard port (e.g. 8080, 3000), include it in the value: `const PRIMARY_BACKEND = "1.2.3.4:8080";`
- **TLS on primary**: the script assumes HTTP for the primary backend. If your self-hosted server has a valid TLS certificate, change `BACKEND_SCHEME` logic to use `"https"`. If it uses a self-signed certificate, Cloudflare will reject the connection unless the SSL mode is set to **Full** (not Full strict) in Cloudflare's SSL settings.
- **Firewall**: make sure your router and firewall allow inbound connections from Cloudflare's IP ranges on the port your self-hosted server uses.
- **Scaling beyond the free tier**: Cloudflare Workers' free plan covers 100,000 requests per day. If you exceed that, the paid plan starts at $5/month for 10 million requests, which remains very affordable.
