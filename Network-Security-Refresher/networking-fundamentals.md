# Networking Fundamentals and Protocol Security

Essential networking concepts, IP addressing, TCP/UDP headers, and protocol security implications.

---

## OSI Model Overview

1. **Layer 7 - Application**: HTTP, HTTPS, SSH, DNS.
2. **Layer 4 - Transport**: TCP (connection-oriented, reliable transmission), UDP (connectionless, low-latency transmission).
3. **Layer 3 - Network**: IP (IPv4 / IPv6), ICMP, packet routing.
4. **Layer 2 - Data Link**: Ethernet, MAC addressing, ARP.

---

## Security Implications

- **TCP SYN Floods**: Exploits the 3-way handshake (`SYN` -> `SYN-ACK` -> `ACK`) by saturating half-open connection queues.
- **Header Injection**: Manipulating `X-Forwarded-For` or `Host` headers to bypass IP-based access controls or trigger HTTP cache poisoning.
- **DNS Cache Poisoning**: Injecting false IP mappings into DNS resolvers to hijack target traffic.
