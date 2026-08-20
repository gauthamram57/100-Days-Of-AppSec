# ⚡ Cross-Site Scripting (XSS) Playbook

A comprehensive guide to Cross-Site Scripting (XSS) types, DOM sources & sinks, WAF bypass methodologies, and payload structures.

---

## 📌 XSS Classifications

### 1. Reflected XSS
- **Mechanism**: Attacker-controlled input is immediately reflected in the HTTP response body without proper encoding/sanitization.
- **Delivery**: Delivered via malicious links containing URL parameters.

### 2. Stored XSS (Persistent XSS)
- **Mechanism**: Payload is stored in backend data stores (e.g., blog comments, user profiles) and executed whenever victims render the stored data.
- **Impact**: Affects any user viewing the page; often used for account takeover or worm propagation.

### 3. DOM-Based XSS
- **Mechanism**: The vulnerability exists entirely client-side. JavaScript reads data from an attacker-controlled **source** (e.g., `location.search`) and passes it to a dangerous **sink** (e.g., `document.write`, `eval()`, `.innerHTML`).
- **Key Note**: The server response might be benign; execution occurs in the DOM parser.

---

## 🎯 Common DOM Sources & Sinks

| Sources (Attacker-Controlled Data) | Dangerous Sinks (Execution Points) |
| :--- | :--- |
| `location.search` | `document.write()` / `document.writeln()` |
| `location.hash` | `element.innerHTML` |
| `location.pathname` | `eval()` |
| `document.referrer` | `setTimeout()` / `setInterval()` |
| `window.name` | `location.href` / `location.assign()` |

---

## 🛡️ Defenses & Mitigation

- **Context-Aware Encoding**: Apply HTML entity encoding, JavaScript string escaping, or URL encoding depending on reflection context.
- **Content Security Policy (CSP)**: Enforce strict `default-src 'self'` and use cryptographic nonces (`nonce-...`) for scripts.
- **Use Safe Sinks**: Use `textContent` instead of `innerHTML`.
- **HTTPOnly Cookies**: Protect session tokens from JavaScript access via `HttpOnly` flag.
