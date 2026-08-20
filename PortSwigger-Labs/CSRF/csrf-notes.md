# 🔄 Cross-Site Request Forgery (CSRF) Playbook

A comprehensive guide to Cross-Site Request Forgery (CSRF) vulnerabilities, attack prerequisites, SameSite cookie attributes, and defense strategies.

---

## 📌 CSRF Mechanics

CSRF occurs when a malicious website causes a victim's web browser to perform an unwanted action on a trusted site where the user is currently authenticated.

```
1. Victim logs into bank.com (Session cookie stored in browser)
2. Victim visits attacker-controlled site (evil.com)
3. evil.com submits hidden form / POST request targeting bank.com/transfer
4. Browser automatically attaches bank.com session cookie
5. bank.com processes valid authenticated request on victim's behalf
```

---

## 🎯 CSRF Prerequisites

1. **Relevant Action**: An action within the application that changes state or sensitive user data (e.g., email update, password reset, funds transfer).
2. **Cookie-Based Session Handling**: The application relies exclusively on browser cookies to authenticate requests.
3. **No Unpredictable Parameters**: The request parameters are entirely predictable by an attacker (no anti-CSRF tokens).

---

## 🛡️ Mitigation Strategies

- **Anti-CSRF Tokens**: Include unique, unpredictable, secret tokens tied to the user's session in state-changing requests.
- **SameSite Cookie Attribute**: Set `SameSite=Strict` or `SameSite=Lax` on sensitive cookies.
- **Re-Authentication**: Require password re-entry or 2FA confirmation for critical account changes.
