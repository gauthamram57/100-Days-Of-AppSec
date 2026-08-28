# Hack The Box (HTB) Writeups 🎯

Welcome to my **Hack The Box Writeups** repository! This repository contains detailed walkthroughs, vulnerability analyses, and proof-of-concept writeups for HTB machines and challenges automatically synchronized from Notion.

---

## 📌 Writeup Index

| Machine / Challenge | OS | Difficulty | Target IP | Writeup Link |
| :--- | :--- | :--- | :--- | :--- |
| **Cap** | Linux | Easy | `10.129.81.72` | [Cap Write-up](./Cap/README.md) |
| **Fireflow** | Linux | Medium | `10.129.81.75` | [Fireflow Write-up](./Fireflow/README.md) |

---

## 🛠️ Machine Breakdown

### 1. [Cap](./Cap/README.md)
* **OS:** Linux
* **Difficulty:** Easy
* **Vector Summary:** Web application security analysis, IDOR/broken access control leading to pcap capture retrieval, credential extraction, and Linux privilege escalation via capabilities (`cap_setuid`).

### 2. [Fireflow](./Fireflow/README.md)
* **OS:** Linux
* **Difficulty:** Medium
* **Vector Summary:** SSH initial access, MCP configuration discovery, JWT authentication bypass, Langflow RCE (CVE-2026-41651), and Kubernetes privileged container breakout (`/host/root`).

---

## 👤 Author

**Gautham Ram**
- GitHub: [@gauthamram57](https://github.com/gauthamram57)

---
*Synchronized automatically from Notion.*
