# 📖 Reading 11 — Web Edge Hardening: TLS, Headers, Rate Limiting, WAF

> **Self-study material for Bonus Lab 11.** Read before starting the lab.

---

## Why an Edge Layer at All

Every public-facing service eventually grows three concerns the application itself isn't well-suited to handle:

1. **TLS termination + cipher posture** — managing certs, rotation, modern protocol versions, OCSP stapling. You don't want this in your app process.
2. **Cross-cutting security headers** — HSTS, CSP, Permissions-Policy. Setting them in every controller route is fragile; setting them once at the edge is durable.
3. **Abuse control** — rate limits, connection caps, IP-based blocking. Your app code shouldn't care about request #1001 from a single IP in a minute; the edge should reject it before the request reaches the upstream.

The pattern is **one Nginx (or Envoy / Caddy / Traefik) in front of N application services**. The app stays focused on business logic; the edge handles security posture.

> 💬 *"Defense in depth lives at the edge. The app is the inner ring; the edge is the outer ring; if either ring is wide-open, you've got a problem."* — paraphrased from Mike Bailey's *Application Layer Network Security* (No Starch, 2024)

---

## TLS in 2026: What "Modern" Means

Lecture 7 introduced TLS misconfiguration as part of IaC scanning (Checkov rule `CKV_AWS_103`). Reading 11 goes deeper.

### Protocol versions

| Version | Status (2026) | Use? |
|---------|---------------|------|
| SSL 2.0/3.0 | Compromised (POODLE, etc.) | ❌ Never |
| TLS 1.0 | Deprecated, PCI-DSS prohibits | ❌ Never |
| TLS 1.1 | Deprecated | ❌ Never |
| TLS 1.2 | Still common in 2026 | ⚠️ Allow only for legacy clients |
| TLS 1.3 | Current best | ✅ Default and preferred |
| TLS 1.4 / QUIC | Draft / emerging | 🧪 Experimental for now |

**Modern profile (Mozilla, 2024+):** TLS 1.3 only. If you need legacy support, TLS 1.2 with a restricted cipher list.

### TLS 1.3 cipher suites

TLS 1.3 reduced the cipher zoo from hundreds to five. You enable them with:

```nginx
ssl_ciphers TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256;
```

`ssl_prefer_server_ciphers off;` — TLS 1.3 ignores server preference; the client picks.

### Elliptic curves

```nginx
ssl_ecdh_curve X25519:secp384r1;
```

X25519 is the modern default (faster + no NSA backdoor concerns vs older NIST curves). secp384r1 is the FIPS-compliant fallback.

### Session resumption

```nginx
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 1d;
ssl_session_tickets off;       # disable to avoid leaking session-key material
```

Resumption skips the TLS handshake on subsequent connections from the same client — major performance win for keep-alive-heavy traffic. Disable session **tickets** (different from session **cache**) because the tickets contain key material that, if leaked, breaks forward secrecy retroactively.

### OCSP stapling

```nginx
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 1.1.1.1 valid=300s;
resolver_timeout 5s;
```

OCSP (Online Certificate Status Protocol) checks whether a certificate has been revoked. Without stapling, every TLS handshake requires the client to query the CA's OCSP responder — adding latency and a privacy leak (CA sees who's connecting where). **Stapling** has the server query OCSP periodically and attach the response to the handshake, eliminating the round-trip.

> ⚠️ **OCSP must-staple:** A cert with the must-staple extension REFUSES connection if stapling fails. Strong defense; also a foot-gun if your resolver is unreachable.

### Cert rotation runbook

The single most-skipped operational concern. Real production runbook:

1. **Monitor expiry** — alert at 30 days, page at 7 days. Most outages are forgotten renewals.
2. **Order/renew** — Let's Encrypt + certbot for free certs; vendor for EV/specialty.
3. **Validate** — `openssl x509 -in newcert.pem -text` shows the new cert; `openssl verify -CAfile ca.pem newcert.pem` validates the chain.
4. **Atomic swap** — symlinks let you swap without an Nginx restart: `ln -sf newcert.pem current.pem && nginx -s reload`.
5. **Verify in production** — `curl -vk https://yoursite.com | head -1` shows the new cert; `testssl.sh` confirms the full posture.
6. **Rollback plan** — keep the previous cert+key on disk for ~7 days. Roll back by re-pointing the symlink.
7. **Audit** — log the rotation event with cert serial + expiry to SIEM/DefectDojo.

Bonus: if you're rotating > 100 certs/yr, automate with cert-manager (K8s) or step-ca (general).

---

## Security Headers: Why They Matter

Every header below defends against a specific bug class. They're free if Nginx adds them; nearly impossible to add reliably in every app route.

### Strict-Transport-Security (HSTS)

```nginx
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

**Defends against:** SSL-stripping attacks where an attacker on the network downgrades a user's connection from HTTPS to HTTP. After the first HSTS-bearing response, browsers refuse HTTP for that origin until the max-age expires.

- `max-age=63072000` = 2 years. Industry standard.
- `includeSubDomains` — propagates to `*.yourdomain.com`. Risky without a full subdomain audit.
- `preload` — opt into Chrome/Firefox's hardcoded HSTS preload list. **Read [hstspreload.org](https://hstspreload.org) before adding** — irreversible.

### X-Content-Type-Options

```nginx
add_header X-Content-Type-Options "nosniff" always;
```

**Defends against:** MIME-sniffing attacks where the browser guesses content-type and executes user-controlled HTML as scripts. With `nosniff`, browsers honor the Content-Type header exactly.

### X-Frame-Options

```nginx
add_header X-Frame-Options "DENY" always;
```

**Defends against:** Clickjacking — embedding your site in an attacker's iframe + tricking users into clicking buttons. DENY refuses framing entirely. SAMEORIGIN allows framing only from your own domain. (Modern preference is CSP's `frame-ancestors` which superseded X-Frame-Options.)

### Referrer-Policy

```nginx
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
```

**Defends against:** Information leakage via the `Referer` header (browser sends the previous URL to the next site). Mode `strict-origin-when-cross-origin` sends just `https://yourdomain.com/` (not the full path) when navigating off-site over HTTPS, and nothing over HTTP.

### Permissions-Policy (was Feature-Policy)

```nginx
add_header Permissions-Policy "camera=(), microphone=(), geolocation=(), payment=()" always;
```

**Defends against:** Third-party JavaScript silently using browser APIs (camera, mic, location). Empty allow-list = nobody. If your app needs the camera, allow `self`.

### Content-Security-Policy (CSP)

The most powerful and most complex header. Defends against XSS, data exfiltration, mixed content, etc.

```nginx
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'nonce-XYZ'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; connect-src 'self'; frame-ancestors 'none'; base-uri 'self'; form-action 'self';" always;
```

Each directive restricts a specific resource type:

- `default-src 'self'` — fallback: only same-origin
- `script-src 'self' 'nonce-XYZ'` — scripts from same origin OR with a server-generated nonce (allows inline scripts you control without `unsafe-inline`)
- `style-src 'self' 'unsafe-inline'` — CSS from same origin + inline styles (still common in 2026)
- `img-src 'self' data: https:` — images from same origin, data URIs, any HTTPS
- `frame-ancestors 'none'` — disallow embedding (replaces X-Frame-Options)
- `base-uri 'self'` — prevents `<base>`-tag hijacking
- `form-action 'self'` — limits where forms can POST

**Strategy:** start with `default-src 'self'` only, deploy in `Content-Security-Policy-Report-Only` mode (logs violations without blocking), iterate. Real apps need weeks of refinement.

---

## Rate Limiting + Connection Limits

Nginx ships two primitives:

### `limit_req` — requests per unit time

```nginx
http {
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    limit_req_status 429;            # default is 503; 429 is more honest
}

server {
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        proxy_pass http://upstream;
    }
}
```

- `$binary_remote_addr` keys by client IP
- `zone=api:10m` reserves 10MB of shared memory (~160k unique IPs)
- `rate=10r/s` = 10 requests per second baseline
- `burst=20` = allow up to 20 queued requests (smooths spikes)
- `nodelay` = serve queued requests immediately instead of throttling

### `limit_conn` — concurrent connections

```nginx
http {
    limit_conn_zone $binary_remote_addr zone=conn:10m;
}

server {
    limit_conn conn 50;
}
```

Caps concurrent connections per IP. Stops slowloris-style and connection-flood attacks.

### Why both?

- `limit_req` is about request volume — "stop pounding /api/login with credential-stuffing attempts."
- `limit_conn` is about connection persistence — "you can't keep 1000 connections open to me."

Different attacks need different limits. Mature configs include both, with separate zones for `/api/login` (tighter), `/api/products` (looser), and `/static/` (essentially unlimited).

### Timeout posture

```nginx
client_body_timeout 10s;
client_header_timeout 10s;
proxy_read_timeout 30s;
proxy_connect_timeout 5s;
send_timeout 10s;
keepalive_timeout 75s;
```

Every timeout is a **fail-closed** behavior. Without them, slowloris attacks (one request, sent one byte per second) tie up workers indefinitely. With them, Nginx returns 408 and frees the worker after the timeout.

---

## WAF: The Next Layer

This course teaches the **OSS-only** edge layer. Real production at scale adds a WAF (Web Application Firewall):

| Tool | Where it runs | Cost |
|------|---------------|------|
| **ModSecurity + OWASP CRS** | In Nginx (mod_security3) | Free (manage yourself) |
| **Cloudflare WAF / AWS WAF / Akamai** | At the CDN edge | $$ |
| **AWS Shield Advanced** | AWS edge, DDoS-focused | $$$ |

ModSecurity + OWASP Core Rule Set (CRS) gives you:
- ~250 rules for OWASP Top 10 patterns (SQL injection, XSS, CSRF, path traversal)
- Per-rule scoring + paranoia levels
- Anomaly scoring (a request with 5 medium-severity rules firing = block)

It's free, but it's also a part-time job to tune. CRS at paranoia level 4 will false-positive on legitimate traffic. Most orgs run CRS at paranoia 1 or 2 in **DetectionOnly** mode for weeks, tune, then escalate.

---

## fail2ban: The Discipline Layer

Nginx logs every request. fail2ban watches the logs and bans IPs that match attack patterns.

```ini
# /etc/fail2ban/jail.d/nginx-auth.conf
[nginx-auth]
enabled = true
filter = nginx-auth
action = iptables-multiport[name=nginx-auth, port="http,https"]
logpath = /var/log/nginx/access.log
maxretry = 5
findtime = 600
bantime = 3600
```

The default filters cover common attacks. Custom filters let you match anything in your access log — e.g., 30 404s in 60 seconds = likely scanner = ban for 1 hour.

This isn't WAF-grade — fail2ban doesn't *prevent* the first attack, only the repeat ones. But it's free, lightweight, and shaves 80% of the noise from your access logs.

---

## Cert Automation: Let's Encrypt + certbot

Free CA-issued certs in 2026 are standard. The flow:

```bash
sudo certbot --nginx \
  -d yourdomain.com \
  -d www.yourdomain.com \
  --agree-tos --email admin@yourdomain.com

# Auto-renewal runs via systemd timer (default since certbot 2.x)
systemctl status certbot.timer
```

Certbot edits your Nginx config to add the cert, and configures a renewal hook that runs every 12 hours. Real-world reliability: typically rotates 30 days before expiry; failures fire emails; works fine for ~99% of sites.

**For multi-domain / wildcard certs:** DNS-01 challenge (instead of HTTP-01). Requires DNS API access for your provider. Worth the setup for any site with subdomain proliferation.

---

## When to Replace Nginx

Nginx is the right answer for 95% of small-to-medium deployments. You'd replace it with:

- **Envoy** — when you need fine-grained traffic shaping, observability, service mesh integration (Istio's data plane uses Envoy)
- **Traefik** — when you want config from Kubernetes annotations / consul / etcd rather than static files
- **Caddy** — when you want default-HTTPS-everywhere with Let's Encrypt baked in
- **HAProxy** — when you need extreme TCP/HTTP performance at the load-balancer layer

For Lab 11 — **stay with Nginx**. It's the de-facto standard; understanding it transfers to all the others.

---

## Operational Patterns Worth Knowing

### Blue/green at the edge

Two upstream servers; switch traffic with a single config-reload:

```nginx
upstream backend {
    server app-blue:3000 weight=100;
    # server app-green:3000 weight=0;   # uncomment to switch
}
```

A `nginx -s reload` is zero-downtime; works for blue/green deploys + canary by adjusting weights.

### Health checks

Nginx OSS does **passive** health checks (mark a backend unhealthy after N failed requests). For **active** health checks (Nginx polling `/health` periodically) you need Nginx Plus or to fall back to Envoy/HAProxy.

```nginx
upstream backend {
    server app1:3000 max_fails=3 fail_timeout=30s;
    server app2:3000 max_fails=3 fail_timeout=30s;
    server app3:3000 backup;
}
```

`backup` servers receive no traffic unless all primaries are down — useful for cross-region fallback.

### Access log analysis

```bash
# Top 20 requesters (excluding monitoring)
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -20

# Slow requests (>1s response time — needs `$request_time` in log_format)
awk '$10 > 1 {print}' /var/log/nginx/access.log | head

# 4xx/5xx by endpoint
awk '$9 >= 400 {print $7, $9}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
```

Three one-liners cover 80% of "what's happening at my edge?"

---

## Resources for Going Deeper

| 📖 Resource | ✍️ Why |
|---|---|
| *Nginx Cookbook* — Derek DeJonghe (O'Reilly, 2nd ed., 2024) | Recipe-style; ch. 9 *"Securing"* is gold |
| *Mozilla Server Side TLS* — [https://wiki.mozilla.org/Security/Server_Side_TLS](https://wiki.mozilla.org/Security/Server_Side_TLS) | Updated continuously; the canonical TLS-config reference |
| *SSL Labs Server Test* — [https://www.ssllabs.com/ssltest/](https://www.ssllabs.com/ssltest/) | Free A-F grade for any public domain |
| *testssl.sh* — [https://testssl.sh/](https://testssl.sh/) | Same test as SSL Labs but as a CLI tool you can run against internal hosts |
| *OWASP Secure Headers Project* — [https://owasp.org/www-project-secure-headers/](https://owasp.org/www-project-secure-headers/) | Per-header defenses + recommended values |
| *Nginx official docs* — [https://nginx.org/en/docs/](https://nginx.org/en/docs/) | Reference for every directive (terse, accurate) |

**For the WAF deep-dive:** *Web Application Defender's Cookbook* — Ryan Barnett (Wiley, 2013) — older but still the practical reference on ModSecurity + OWASP CRS.

**For TLS internals:** *Bulletproof SSL and TLS* — Ivan Ristic (2nd ed., 2024). The most exhaustive book on TLS posture; pairs perfectly with this reading.

---

## How This Reading Maps to Lab 11

| Lab 11 task | This reading section |
|---|---|
| Task 1.1 TLS 1.3 + SSL config | "TLS in 2026" — protocol versions + cipher suites + curves + session resumption |
| Task 1.2 Security headers | "Security Headers" — all six required |
| Task 1.3 Rate + connection limits | "Rate Limiting" |
| Task 1.4 Timeouts | "Timeout posture" |
| Task 2 Cipher hardening + OCSP + rotation | "TLS in 2026" + "Cert rotation runbook" |

Read this first. Then attempt Lab 11. Re-read sections when you hit a pitfall.

> 💬 *"Get TLS and headers right at the edge; everything inside the perimeter inherits the gain."* — paraphrased standing advice from every AppSec consultant.
