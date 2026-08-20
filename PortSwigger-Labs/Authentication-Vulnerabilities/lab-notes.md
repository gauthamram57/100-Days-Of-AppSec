# 📝 Authentication Vulnerabilities Lab Walkthroughs

Detailed walkthroughs and step-by-step solutions for PortSwigger Web Security Academy Authentication labs.

---

### Lab 1: Username Enumeration via Subtly Different Responses
* **Objective**: Enumerate valid username and brute-force password.
* **Method**:
  1. Captured login request in Burp Suite and sent to **Intruder** (Sniper attack).
  2. Applied **Grep-Extract** to match response text (`Invalid username` vs `Incorrect password`).
  3. Identified valid username from response difference.
  4. Fixed username and brute-forced password until receiving a `302 Found` redirect.

---

### Lab 2: Username Enumeration via Response Timing
* **Objective**: Enumerate username using server response latency.
* **Method**:
  1. Added `X-Forwarded-For` header to bypass basic IP checks.
  2. Used a **Pitchfork attack** in Intruder with varying usernames and a long password payload.
  3. Evaluated server response time: requests with valid usernames took significantly longer to process due to password hashing execution.
  4. Brute-forced password while rotating `X-Forwarded-For` IP headers until receiving a `302 Found`.

---

### Lab 3: Broken Brute-Force Protection, IP Block
* **Objective**: Bypass IP-based failed login limit.
* **Method**:
  1. Observed that IP gets blocked after 3 failed login attempts.
  2. Configured Intruder **Pitchfork attack**.
  3. Interleaved a successful login attempt (`wiener:peter`) after every 2 `carlos` attempts.
  4. The successful login reset the failed-attempt counter on the server, enabling full password brute-force completion (`carlos:159753`).

---

### Lab 4: Brute-Forcing a Stay-Logged-In Cookie
* **Objective**: Crack persistent session cookie for target user.
* **Method**:
  1. Captured `stay-logged-in` cookie from valid login: Base64 decoded to reveal `username:MD5(password)`.
  2. Sent `GET /my-account?id=carlos` to Intruder.
  3. Configured Payload Processing: `MD5 Hash` → `Add Prefix "carlos:"` → `Base64 Encode`.
  4. Identified valid candidate password returning `200 OK`.

---

### Lab 5: Offline Password Cracking via XSS Cookie Theft
* **Objective**: Steal target user's stay-logged-in cookie and crack password offline.
* **Method**:
  1. Injected stored XSS into blog comment section to exfiltrate victim cookies to Exploit Server access log.
  2. Captured Carlos's `stay-logged-in` cookie.
  3. Base64 decoded cookie to retrieve MD5 hash and cracked it offline.
