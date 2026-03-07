# Lo-Fi — TryHackMe Writeup

**Difficulty:** Easy
**Category:** Web
**Vulnerability:** Local File Inclusion (LFI)

---

## Overview

Lo-Fi is a simple web application hosting lofi music playlists. The page loads content dynamically through a `page` parameter in the URL, which turns out to be vulnerable to Local File Inclusion.

---

## Reconnaissance

Navigating to the target, the URL immediately stood out:

```
http://<TARGET_IP>/?page=relax.php
```

The `page` parameter is loading a PHP file directly by name — a classic indicator of LFI.

---

## Exploitation

### Testing for LFI

By replacing the `page` value with directory traversal sequences, it's possible to read arbitrary files on the server:

```
http://<TARGET_IP>/?page=../../../../etc/passwd
```

The server returned the contents of `/etc/passwd`, confirming the vulnerability.

### Retrieving the Flag

With LFI confirmed, the flag was found at the root of the filesystem:

```
http://<TARGET_IP>/?page=../../../../flag.txt
```

---

## Key Takeaways

- **LFI** occurs when user input is passed directly to file inclusion functions without proper sanitization.
- Directory traversal (`../`) allows navigating outside the intended directory.
- Always check URL parameters that reference filenames — they are common LFI entry points.

---

## Vulnerability Summary

| Field | Detail |
|-------|--------|
| Type | Local File Inclusion (LFI) |
| Parameter | `?page=` |
| Impact | Arbitrary file read on the server |
| Fix | Whitelist allowed filenames; never pass raw user input to file functions |
