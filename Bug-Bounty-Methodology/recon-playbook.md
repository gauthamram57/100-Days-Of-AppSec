# Bug Bounty Reconnaissance Playbook

A methodology for subdomain enumeration, asset discovery, port scanning, and HTTP probing during security assessments.

---

## Subdomain Enumeration

### 1. Passive Subdomain Discovery
Execute multiple passive enumeration tools to discover the target attack surface without directly probing target infrastructure:

```bash
# Subfinder - Fast passive subdomain discovery
subfinder -d target.com -all -o subfinder_out.txt

# Assetfinder - Find domains and subdomains related to target
assetfinder --subs-only target.com > assetfinder_out.txt

# Amass - In-depth OSINT subdomain enumeration
amass enum -passive -d target.com -o amass_out.txt
```

### 2. Merging and Deduplication
Combine and deduplicate all discovered subdomains:

```bash
cat subfinder_out.txt assetfinder_out.txt amass_out.txt | sort -u > all_subdomains.txt
```

---

## Live Host Probing (`httpx`)

Probe discovered subdomains for active HTTP/HTTPS web servers, status codes, page titles, and web technologies:

```bash
httpx -l all_subdomains.txt -title -status-code -tech-detect -follow-redirects -o live_web_hosts.txt
```

---

## Attack Surface Prioritization

1. **Non-Standard Ports**: Inspect web services running on ports `8080`, `8443`, `8000`, `8888`, `9000`.
2. **Admin and Dev Endpoints**: Filter for keywords (`admin`, `dev`, `staging`, `test`, `api`, `internal`).
3. **Third-Party Services**: Identify misconfigured CNAME records pointing to unclaimed services (Subdomain Takeover).
