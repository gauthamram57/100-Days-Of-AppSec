# 🛡️ 100-Days-Of-AppSec

Documenting my daily journey of mastering Application Security, web exploitation, DevSecOps, and PortSwigger Web Security Academy labs.

---

## 📁 Repository Structure & Notes Index

Every vulnerability topic is organized into a standardized 3-file structure:
1. **`<attack-type>-notes.md`**: Core theoretical concepts, attack taxonomy, and mitigation strategies.
2. **`lab-notes.md`**: Practical lab walkthroughs, step-by-step solutions, and payload analysis.
3. **`<attack-type>-automation-script.md`**: Custom Python scripts, PoCs, and automation templates.

```
PortSwigger-Labs/
├── SQL-Injection/
│   ├── sqli-notes.md               # SQLi Playbook & Attack Vectors
│   ├── lab-notes.md                # PortSwigger SQLi Lab Solutions
│   └── sqli-automation-script.md   # Python Blind SQLi Automation Script
├── Authentication-Vulnerabilities/
│   ├── authentication-notes.md     # Auth Factors & Flaw Mechanics
│   ├── lab-notes.md                # Auth Lab Walkthroughs (Timing, Interleaving, Cookies)
│   └── authentication-automation-script.md # Cookie Brute-Force Script
├── Cross-Site-Scripting-XSS/
│   ├── xss-notes.md                # Reflected, Stored & DOM XSS Playbook
│   ├── lab-notes.md                # XSS Labs (Custom Tags, SVG, WAF Bypasses)
│   └── xss-automation-script.md    # Cookie Exfiltration & JS Payloads
└── CSRF/
    ├── csrf-notes.md               # CSRF Mechanics & Prerequisites
    ├── lab-notes.md                # CSRF Lab Walkthroughs
    └── csrf-poc-generator.md       # Auto-Submitting HTML PoC Template
```

---

## 📜 License

This repository is licensed under the [MIT License](LICENSE).
