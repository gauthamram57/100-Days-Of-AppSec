# 🔐 Authentication Vulnerabilities Playbook

A comprehensive reference for authentication mechanisms, multi-factor authentication concepts, rate-limiting bypasses, and credential brute-forcing.

---

## 🧠 Core Authentication Concepts

- **Authentication**: The process of verifying that a user is who they claim to be.
- **Authorization**: Determining whether an authenticated user is permitted to perform a specific action or access a resource.

### Factors of Authentication
1. **Knowledge Factors**: Something you know (e.g., passwords, security questions, PINs).
2. **Possession Factors**: Something you have (e.g., security tokens, mobile authenticator apps, SMS OTPs).
3. **Inherence Factors**: Something you are (e.g., biometrics, fingerprints, behavioral patterns).

---

## ⚡ Common Authentication Weaknesses

- **Weak Password Policies**: Failure to enforce strong passwords, enabling simple dictionary attacks.
- **Lack of Rate Limiting**: Allowing unlimited automated login attempts.
- **Flawed Rate Limiting**: Limiting attempts solely per IP address or per username without cross-bound protection.
- **Verbose Error Messages**: Differentiating error messages between invalid usernames and invalid passwords.
- **Predictable Session Tokens**: Using simple hashes or encodings (e.g., `Base64(username:MD5(password))`) for persistent session cookies ("Stay Logged In").

---

## 🛡️ Defenses & Mitigation

- Enforce unified, non-descript error messages (`Invalid username or password`).
- Implement IP-based rate limiting paired with account lockout thresholds.
- Enforce secure session token generation using cryptographically secure random values (e.g., UUIDv4 / HMAC-SHA256).
