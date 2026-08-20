# 🌐 HTTP Request Lifecycle & Security Architecture

*Full notes imported from Notion.*

## 📌 The Big Picture

A request from your browser doesn't hit a single "server" — it hits a chain:

```
Browser ──> DNS ──> Load Balancer (LB) / CDN ──> Reverse Proxy (Nginx, Cloudflare) ──> Web Server ──> Application Code ──> ORM ──> Database
```

> **Key Concept**: Auth/session checks and caching don't live in one box — they sit at multiple points across this chain (proxy, app layer, DB layer can each do their own check; browser cache, CDN cache, app cache, DB cache can each be stale independently).

Why draw it as a cycle, not a straight line? **Every layer gets touched twice** — once on the way in, once on the way back out with the response. Most annoying bugs (missing headers, stale data, broken cookies) live on the return trip, not the forward one.

---

## 1. Browser

Where it starts. You type a URL / click a link / JavaScript fires a `fetch()`. Needs an IP address before it can do anything else — hence DNS is next.

### Security & Attacks at Browser Layer
- **XSS (Cross-Site Scripting)**: Attacker sneaks malicious JavaScript into a page you trust (e.g., an unsanitized comment box). When your browser runs it, it can steal your session cookie or act as you.
- **CSRF (Cross-Site Request Forgery)**: You're logged into your bank in one tab; a malicious page in another tab tricks your browser into submitting a request to the bank using your still-valid session cookie.

---

## 2. DNS (Domain Name System)

Translates a domain name (`example.com`) into an IP address because machines route by IP, not by name. If DNS is slow or compromised, downstream components never receive legitimate traffic.

### Security & Attacks at DNS Layer
- **DNS Spoofing / Cache Poisoning**: Attacker tricks a resolver into caching a wrong IP for a domain, so typing `bank.com` sends you to the attacker's server instead.
- **DNS Hijacking**: Attacker compromises your domain registrar account and changes where your domain points, redirecting all traffic.

---

## 3. Load Balancer (LB) vs. CDN

### Load Balancer (LB)
Distributes traffic across multiple copies of your own servers (e.g. 5 identical servers running your app), so no single server gets overwhelmed. Purely about traffic distribution — no caching or physical geography involved.

### CDN (Content Delivery Network)
A global network of servers, physically closer to users than your origin server, that caches copies of your content (images, JS, CSS, sometimes whole pages).

- **Cache Hit**: CDN edge server answers immediately from its own copy. Origin server never sees this request.
- **Cache Miss**: CDN fetches once from origin, saves a copy, and serves the user. Next nearby user gets the fast cached version.

> **Distinction**: LB spreads traffic among your own servers in one place. CDN spreads copies of your content across the planet. Large setups use both — CDN out front globally, LB behind it distributing whatever reaches origin across multiple application servers.

---

## 4. CDN Origin IP Exposure & CDN Bypass

### The Origin/CDN Split Attack Surface
1. **Origin IP Exposure**: If an attacker finds your real origin IP (via old DNS `A` records prior to CDN setup, forgotten subdomains, or DNS history tools like SecurityTrails), they can attack that IP directly, completely bypassing CDN WAF protection.
2. **SSL Termination Trust**: With modes like Cloudflare's "Flexible SSL," HTTPS ends at the CDN edge, meaning the CDN can technically see traffic in plaintext before re-encrypting toward origin. Stricter modes ("Full (strict)") keep the origin connection encrypted and verified.
3. **CDN Account Security**: If an attacker compromises your CDN dashboard login, they can redirect all traffic, issue fake certificates, or disable protections without touching your origin server.

### Origin Firewall Mitigation
Configure the origin server's firewall (`iptables` / cloud security group) to accept connections **only from the CDN's published IP ranges**, rejecting everything else — so even if the real origin IP leaks, it cannot be reached directly.

---

## 5. DDoS via Leaked Origin IP

### How it Works
* **DDoS (Distributed Denial of Service)**: Flooding a server with far more requests than it can handle (connections, bandwidth, CPU) until it crashes.
* Behind a CDN, floods are absorbed at the CDN's edge. But if an attacker points a botnet straight at a leaked origin IP, they skip that shield entirely.

```
Attacker ──(Botnet Flood)──> Leaked Origin IP (Bypasses CDN WAF Shield) ──> Origin Server Crash
```
