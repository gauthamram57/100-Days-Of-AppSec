# HTTP Request Lifecycle and Security Architecture

## Overview

A request from your browser passes through multiple architecture layers:

```
Browser -> DNS -> Load Balancer (LB) / CDN -> Reverse Proxy (Nginx, Cloudflare) -> Web Server -> Application Code -> ORM -> Database
```

Authentication, session verification, and caching sit at multiple points across this chain (proxy, application layer, and database layer can each perform independent checks).

Every layer is touched twice: once on the incoming request, and once on the outgoing response. Diagnostic issues (missing headers, stale cache, broken cookies) frequently occur on the return path.

---

## 1. Browser Layer

Where the request originates (URL navigation, link clicks, or `fetch()` API execution). The browser requires an IP address before transmitting data.

### Security and Attack Vectors
- **Cross-Site Scripting (XSS)**: Malicious JavaScript is injected into untrusted inputs (e.g., comment fields). When rendered, the browser executes the script in the context of the user session.
- **Cross-Site Request Forgery (CSRF)**: Tricking a logged-in user's browser into submitting unauthorized requests to a target application while carrying valid session cookies.

---

## 2. DNS (Domain Name System)

Translates domain names (`example.com`) into IP addresses. If DNS resolution is compromised, downstream components do not receive legitimate requests.

### Security and Attack Vectors
- **DNS Spoofing / Cache Poisoning**: Tricking a resolver into caching incorrect IP mappings, redirecting users to attacker-controlled infrastructure.
- **DNS Hijacking**: Compromising domain registrar credentials to alter nameserver configuration.

---

## 3. Load Balancer vs. Content Delivery Network (CDN)

### Load Balancer (LB)
Distributes incoming traffic across multiple application instances to prevent single-point overload. Focuses strictly on traffic distribution without edge caching.

### Content Delivery Network (CDN)
A distributed network of edge servers physically closer to end users that caches static and dynamic assets.

- **Cache Hit**: Served directly from the CDN edge server without reaching the origin server.
- **Cache Miss**: The CDN fetches content from the origin server, stores a copy, and serves the user.

---

## 4. CDN Origin IP Exposure and Bypass

### Security Risks
1. **Origin IP Exposure**: Finding the real origin IP (via historical DNS records, unproxied subdomains, or reconnaissance tools) allows attackers to bypass CDN WAF controls.
2. **SSL Termination**: In incomplete TLS modes (e.g., Flexible SSL), traffic between the CDN edge and origin server is unencrypted.
3. **CDN Account Compromise**: Access to the CDN management dashboard allows unauthorized traffic redirection or security rule suppression.

### Mitigation
Configure origin firewalls (`iptables` or cloud security groups) to accept connections exclusively from published CDN IP ranges, dropping all direct traffic.

---

## 5. Distributed Denial of Service (DDoS)

DDoS attacks attempt to saturate target bandwidth, CPU, or connection pools using distributed botnets. While CDNs absorb volumetric traffic at edge locations, direct attacks on exposed origin IPs bypass edge protections.
