# Hack The Box (HTB) Writeups

Welcome to my **Hack The Box Writeups** repository! This directory contains detailed walkthroughs, vulnerability analyses, and proof-of-concept writeups for HTB machines and challenges.

---

## Progress & Statistics

| Total Machines | Easy | Medium | Hard | Primary Focus |
| :---: | :---: | :---: | :---: | :--- |
| **6** | 2 | 3 | 1 | Web Security, API Security, Container/K8s Breakouts, CMS Object Injection, SMB/Spoolss RCE |

---

## Methodology & Framework

Before diving into specific machine writeups, check out the [HTB Machine Methodology Guide](./htb-methodology.md) for a complete breakdown of recon, web exploitation, and privilege escalation patterns.

---

## Writeup Index

| Machine / Challenge | OS | Difficulty | Target IP | Writeup Link |
| :--- | :--- | :--- | :--- | :--- |
| **Cap** | Linux | Easy | `10.129.81.72` | [Cap Write-up](./Cap/README.md) |
| **Fireflow** | Linux | Medium | `10.129.81.75` | [Fireflow Write-up](./Fireflow/README.md) |
| **Nexus** | Linux | Hard | `10.129.234.54` | [Nexus Write-up](./Nexus/README.md) |
| **Cohort** | Linux | Medium | `10.129.82.27` | [Cohort Write-up](./Cohort/README.md) |
| **Orion** | Linux | Easy | `10.129.126.138` | [Orion Write-up](./Orion/README.md) |
| **Abducted** | Linux | Medium | `10.129.244.177` | [Abducted Write-up](./Abducted/README.md) |

---

## Machine Breakdown

### 1. [Cap](./Cap/README.md)
* **OS:** Linux
* **Difficulty:** Easy
* **Vector Summary:** Web application security analysis, IDOR/broken access control leading to pcap capture retrieval, credential extraction, and Linux privilege escalation via capabilities (`cap_setuid`).

### 2. [Fireflow](./Fireflow/README.md)
* **OS:** Linux
* **Difficulty:** Medium
* **Vector Summary:** SSH initial access, MCP configuration discovery, JWT authentication bypass, Langflow RCE (CVE-2026-41651), and Kubernetes privileged container breakout (`/host/root`).

### 3. [Nexus](./Nexus/README.md)
* **OS:** Linux
* **Difficulty:** Hard
* **Vector Summary:** SSH credentials, JWT `none` alg forgery on MCP server, dynamic tool registration RCE, K8s service account token, `nodes/proxy` access, and kubelet WebSocket exec on privileged node-exporter container.

### 4. [Cohort](./Cohort/README.md)
* **OS:** Linux
* **Difficulty:** Medium
* **Vector Summary:** SSRF via `/api/validate` to internal Marimo notebook service (`0.0.0.0/status`), WebSocket terminal RCE, and PackageKit 1.2.8 privilege escalation via Pack2TheRoot (CVE-2026-41651).

### 5. [Orion](./Orion/README.md)
* **OS:** Linux
* **Difficulty:** Easy
* **Vector Summary:** Craft CMS 5.6.16 web enumeration (`/admin/login`), CVE-2025-32432 object injection via asset image-transform (`PhpManager` construct), session poisoning with PHP RCE payload, and initial shell access.

### 6. [Abducted](./Abducted/README.md)
* **OS:** Linux
* **Difficulty:** Medium
* **Vector Summary:** Samba SMB anonymous enumeration (`HP-Reception` printer share), CVE-2026-4480 spoolss RPC command injection (`|sh` document_name & `%J` print command execution), ICMP validation, and reverse shell as `nobody`.

---

## Author

**Gautham Ram**
- GitHub: [@gauthamram57](https://github.com/gauthamram57)

---
*Synchronized automatically from Notion.*
