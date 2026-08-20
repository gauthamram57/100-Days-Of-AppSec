# 🌐 HTTP Request Lifecycle & Architecture Notes

A detailed walkthrough of the end-to-end HTTP request lifecycle from browser interaction to backend server processing.

---

## 🗺️ Path of an HTTP Request

```
[Browser Client] ──(1. DNS Lookup)──> [DNS Server]
       │
       ├──(2. TLS Handshake / TCP 443)──> [CDN Edge (Cloudflare/CloudFront)]
                                                  │
                                          (3. Reverse Proxy / WAF)
                                                  │
                                          (4. Load Balancer)
                                                  │
                                          (5. Application Server - FastAPI/Django)
```

---

## 🔍 Step-by-Step Breakdown

1. **DNS Resolution**: Converts domain (`target.com`) to IP via recursive resolver.
2. **TCP & TLS Handshake**: Establishes encrypted channel (`HTTPS / TLS 1.3`).
3. **WAF & CDN Inspection**: Filters malicious payloads, enforces rate limits, and serves cached static assets.
4. **Load Balancer**: Distributes traffic across backend application instances.
5. **Application Framework Execution**: Routing, middleware processing (auth, CORS), database querying, and HTTP response generation.
