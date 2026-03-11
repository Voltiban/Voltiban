#  Hidden Deep Into my Heart - TryHackMe Writeup

**Difficulty:** Easy | **Category:** Web | **Points:** 100

---

## Summary

A Valentine's Day themed message board hiding a secret admin vault. The vulnerability chain involves information disclosure via `robots.txt` and directory enumeration leading to credential exposure.

---

## Reconnaissance

Started with a quick directory scan:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -u http://TARGET:5000/FUZZ \
     -mc 200,301,302,403 -t 50
```

**Found:** `robots.txt`

---

## Exploitation

### Step 1 — robots.txt disclosure

```
User-agent: *
Disallow: /cupids_secret_vault/
# cupid_arrow_2026!!!
```

Two pieces of information leaked:
- A hidden path: `/cupids_secret_vault/`
- A password in a comment: `cupid_arrow_2026!!!`

### Step 2 — Directory enumeration

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -u http://TARGET:5000/cupids_secret_vault/FUZZ \
     -mc 200,301,302 -t 50
```

**Found:** `/cupids_secret_vault/administrator`

### Step 3 — Login

Navigated to `/cupids_secret_vault/administrator` and tried:

| Username | Password |
|----------|----------|
| admin | cupid_arrow_2026!!! |

✅ Access granted — flag retrieved.

---

## Vulnerability

**Information Disclosure via robots.txt** — The `robots.txt` file is intended to guide search engine crawlers, but developers sometimes accidentally expose sensitive paths and credentials in comments.

---

## Flag

`THM{l0v3_is_in_th3_r0b0ts_txt}`

---

## Key Takeaways

- Always check `robots.txt` — it's one of the first things to look at in web CTFs
- Comments in config files are not secure storage for credentials
- `robots.txt` paradoxically highlights the most sensitive paths on a site
