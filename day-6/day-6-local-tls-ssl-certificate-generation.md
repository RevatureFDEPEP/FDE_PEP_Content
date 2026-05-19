# Local TLS/SSL Certificate Generation

> *Day 6 — Week 2, Monday*

## Overview

Plain HTTP is acceptable for this course's local development, but production never is. Sooner than you expect, a piece of code will refuse to work over HTTP — a browser will block a third-party widget on a secure page from making insecure API calls, a `Secure` cookie will silently not be set, a service worker will refuse to register, or a webhook provider will reject your localhost URL. This file shows you how to generate a local TLS certificate and terminate TLS at the reverse proxy so HTTPS works on your laptop the same way it works in production.

The depth here is deliberate but bounded: you should be able to do this when you need it. You will not be required to ship TLS on the Day 6 deliverable.

## What "terminating TLS at the proxy" means

TLS termination is a deployment pattern: the reverse proxy handles the encrypted side of the connection (decrypts incoming, encrypts outgoing to the client), and backends behind it speak plain HTTP on the internal network. The proxy is the only thing that owns a certificate; the four backend services do not need to know TLS exists.

This is the standard pattern for several reasons:

- One place to manage certificates, ciphers, and TLS version policy.
- Backends do not need TLS libraries, do not need certificates rotated, do not need to be restarted on cert renewal.
- The performance cost of TLS (CPU for the handshake, mainly) is paid once at the edge.
- The internal `backend` network is `internal: true` and unreachable from outside — plain HTTP there is no risk in this topology.

## Two ways to make a local certificate

### Option A — `openssl` (no extra tooling, ugly browser warnings)

The certificate-and-key pair every TLS course starts with:

```bash
# Generate a 2048-bit RSA private key
openssl genrsa -out localhost.key 2048

# Generate a self-signed certificate, valid for 365 days
openssl req -new -x509 -sha256 -key localhost.key -out localhost.crt -days 365 \
    -subj "/CN=localhost" \
    -addext "subjectAltName=DNS:localhost,DNS:*.localhost,IP:127.0.0.1"
```

This works — Nginx will load it, browsers will see HTTPS — but the browser will throw a "your connection is not private" warning every time, because no trusted authority signed the certificate. You can click through, but it is noisy.

### Option B — `mkcert` (one-time setup, no warnings)

[`mkcert`](https://github.com/FiloSottile/mkcert) generates certificates signed by a *local* certificate authority that you install into your system trust store. Browsers then accept those certificates without complaint.

```bash
# Install mkcert (brew, scoop, apt — see the project page)
brew install mkcert            # macOS
scoop install mkcert           # Windows
sudo apt install libnss3-tools mkcert  # Linux

# One-time: install the local CA root into your trust stores
mkcert -install

# Generate a cert for localhost (and any other names you want)
mkcert localhost 127.0.0.1 ::1
# Output: localhost+2.pem (cert), localhost+2-key.pem (key)
```

Now your browser trusts the cert. Other developers on the same machine inherit the trust; CI does not, and that is fine — CI typically does not need a trusted cert.

**For this course, prefer `mkcert`.** It is a one-time install and removes a class of "is this a real error or a self-signed warning?" confusion when debugging.

## Wiring the certificate into Nginx

Drop the two files into the proxy container and tell Nginx where they are:

```yaml
services:
  proxy:
    image: nginx:1.27-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./infra/nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./infra/nginx/certs:/etc/nginx/certs:ro
```

```nginx
# nginx.conf — TLS-enabled server block
http {
    # ... upstreams from the path-routing config ...

    # Redirect plain HTTP to HTTPS
    server {
        listen 80;
        server_name localhost;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        http2 on;
        server_name localhost;

        ssl_certificate     /etc/nginx/certs/localhost.pem;
        ssl_certificate_key /etc/nginx/certs/localhost-key.pem;

        # Sensible defaults — modern TLS only
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers off;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 1d;

        # Same location blocks as the HTTP-only config
        location /api/users/ {
            proxy_pass http://user_service/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
        # ... etc.
    }
}
```

Two server blocks: one on `:80` that does nothing but redirect to HTTPS, and one on `:443` that does the real work. The `X-Forwarded-Proto $scheme` header tells the backend that the original request was HTTPS even though the proxy-to-backend hop is plain HTTP — important for any code that generates absolute URLs in responses (password reset links, OAuth callbacks).

## Verifying it works

```bash
docker compose up -d proxy
curl -v https://localhost/healthz
# With mkcert: 200, no warnings
# With openssl self-signed: add -k to ignore cert verification

curl -v http://localhost/healthz
# 301 redirect to https://localhost/healthz
```

In the browser, navigate to `https://localhost/` — with mkcert installed, the lock icon should appear with no warning.

## When not to terminate TLS at the proxy

Two cases where TLS-only-at-the-proxy is wrong:

1. **End-to-end encrypted requirements** — regulated workloads (PCI, some HIPAA contexts) sometimes require encryption all the way to the backend. Then each backend also runs TLS and the proxy speaks TLS to it (`proxy_pass https://...`).
2. **The backend is on an untrusted network** — if the proxy and backend are in different VPCs or data centers, you cannot assume the inter-service hop is private. Same fix: TLS to the backend.

For this course (proxy and backends in one Compose stack on `internal: true`), neither applies. Terminate at the edge.

## Common Pitfalls

- **Committing the key file to git.** `localhost-key.pem` is a private key. Add `*.pem` and `certs/` to `.gitignore` *before* you generate them, not after.
- **Forgetting the redirect from `:80` to `:443`.** If `:80` keeps serving content, half your traffic stays unencrypted and you may not notice for weeks. The `return 301` server block is non-negotiable.
- **Not setting `X-Forwarded-Proto`.** Backends that generate absolute URLs (e.g., for OAuth redirect URIs) will generate `http://...` and the OAuth flow will fail in subtle ways. Always pass the original scheme.
- **Mixing certificates across environments.** A cert generated for `localhost` will not match `app.staging.example.com`. Each environment needs its own cert (in production, almost always Let's Encrypt or a managed cert from the cloud provider).
- **Expecting `mkcert` certs to work in CI.** Each machine has its own root CA. CI will see a cert it does not trust. Either use `-k` in CI, or generate a cert per environment.

## Key Takeaways

- TLS termination at the proxy is the standard pattern: one cert at the edge, plain HTTP behind it on the internal network.
- For local dev, `mkcert` is the no-friction choice; `openssl` works but you eat browser warnings.
- The two essential headers on TLS-terminating proxies are `X-Forwarded-Proto` and `Host` — without them, backend code that generates URLs will produce wrong output.
- HTTP-to-HTTPS redirect is a separate `server` block. Do not skip it.
- TLS is not required for the Day 6 deliverable, but you should now know what to do when you need it.

---

*Prerequisites: Day 6 — Reverse proxy fundamentals, Day 6 — Path-based routing to microservices.*
