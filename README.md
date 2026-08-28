# 100-Days-Of-AppSec

Documenting my daily journey of mastering Application Security, web exploitation, DevSecOps, container security, bug bounty methodology, networking, Hack The Box (HTB) writeups, and PortSwigger Web Security Academy labs.

---

## Repository Structure and Index

```
100-Days-Of-AppSec/
├── Bug-Bounty-Methodology/
│   ├── recon-playbook.md           # Subdomain Enumeration (subfinder, amass, httpx)
│   └── attack-flow.md              # 5-Step Vulnerability Discovery Framework
├── HTB-Writeups/                   # Hack The Box Walkthroughs & Machine Writeups
│   ├── README.md                   # HTB Index & Machine Breakdown
│   ├── htb-methodology.md          # Unified HTB Penetration Testing Methodology
│   ├── Cap/                        # Cap (Linux / Easy) - IDOR, Pcap Analysis, Capabilities Privesc
│   ├── Fireflow/                   # Fireflow (Linux / Medium) - MCP, JWT Bypass, Langflow RCE, K8s Breakout
│   ├── Nexus/                      # Nexus (Linux / Hard) - JWT alg=none, MCP Tool RCE, K8s Kubelet Exec
│   └── Cohort/                     # Cohort (Linux / Medium) - SSRF, Marimo WebSocket RCE, PackageKit CVE-2026-41651
├── Network-Security-Refresher/
│   ├── request-lifecycle.md        # HTTP Request Lifecycle (DNS -> CDN -> LB -> App)
│   └── networking-fundamentals.md  # OSI Model & Protocol Security
├── DevSecOps-Container-Security/
│   └── docker-kubernetes-notes.md  # Docker CLI, Hardening & Trivy Image Scanning
├── PortSwigger-Labs/
│   ├── SQL-Injection/
│   ├── Authentication-Vulnerabilities/
│   ├── Cross-Site-Scripting-XSS/
│   └── CSRF/
```

---

## Hack The Box Writeups

Detailed walkthroughs for retired HTB machines and custom labs are organized in [`HTB-Writeups/`](./HTB-Writeups/README.md):

| Machine / Challenge | OS | Difficulty | Target IP | Writeup Link |
| :--- | :--- | :--- | :--- | :--- |
| **Cap** | Linux | Easy | `10.129.81.72` | [Cap Write-up](./HTB-Writeups/Cap/README.md) |
| **Fireflow** | Linux | Medium | `10.129.81.75` | [Fireflow Write-up](./HTB-Writeups/Fireflow/README.md) |
| **Nexus** | Linux | Hard | `10.129.234.54` | [Nexus Write-up](./HTB-Writeups/Nexus/README.md) |
| **Cohort** | Linux | Medium | `10.129.82.27` | [Cohort Write-up](./HTB-Writeups/Cohort/README.md) |

---

## License

This repository is licensed under the [MIT License](LICENSE).
