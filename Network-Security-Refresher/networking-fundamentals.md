# 📡 Networking Fundamentals & Security Implications

Essential networking concepts, IP addressing, TCP/UDP headers, and HTTP protocol security.

---

## 📌 OSI Model Layers

1. **Layer 7 - Application**: HTTP, HTTPS, SSH, DNS.
2. **Layer 4 - Transport**: TCP (connection-oriented, reliable), UDP (connectionless, fast).
3. **Layer 3 - Network**: IP (IPv4 / IPv6), ICMP, routing.
4. **Layer 2 - Data Link**: Ethernet, MAC addresses, ARP.

---

## 🔒 Security Implications

- **TCP SYN Floods**: Exploits 3-way handshake (`SYN` → `SYN-ACK` → `ACK`) by saturating half-open connection queues.
- **Header Injection**: Manipulating `X-Forwarded-For` or `Host` headers to bypass rate limits or trigger cache poisoning.
- **DNS Spoofing / Cache Poisoning**: Injecting false IP mappings into DNS resolvers to hijack target traffic.
