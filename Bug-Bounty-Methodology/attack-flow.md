# ⚡ Structured 5-Step Vulnerability Discovery Framework

A systematic 5-step approach for finding and confirming web application logic flaws and injection vulnerabilities.

---

```
Step 1: Find Injection Point
         │
         ▼
Step 2: Test Single Quote / Syntax Probe
         │
         ▼
Step 3: Break Context & Observe Errors
         │
         ▼
Step 4: Verify Payload & Execution
         │
         ▼
Step 5: Extract Data & Demonstrate Impact
```

---

## 🛠️ Step-by-Step Breakdown

### Step 1: Find Injection Point
Identify all attacker-controlled input parameters:
- GET URL parameters (`?id=`, `?search=`, `?cat=`)
- POST body fields (JSON keys, form fields)
- HTTP Headers (`User-Agent`, `Referer`, `X-Forwarded-For`, `Cookie`)

### Step 2: Test Quote & Syntax Probing
Inject special characters (`'`, `"`, `;`, `<`, `>`, `{{`, `${`) to check how the application backend handles raw input.

### Step 3: Break Context & Observe Errors
- Look for HTTP `500 Internal Server Error`, database stack traces, or missing JSON keys.
- Check if response structure changes depending on valid vs invalid syntax.

### Step 4: Verify Payload & Execution
Craft syntax-valid payloads that repair surrounding code (e.g., `' OR 1=1 --`, `"-alert(1)}//`, `&apos;`).

### Step 5: Demonstrate Impact
Extract proof-of-concept data (e.g., `document.cookie`, database version, unauthorized data access) without causing service disruption.
