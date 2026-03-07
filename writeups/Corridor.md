# Corridor — TryHackMe Writeup

**Difficulty:** Easy
**Category:** Web
**Vulnerability:** Insecure Direct Object Reference (IDOR) via MD5 Hashes

---

## Overview

Corridor presents a visual hallway with several doors. Each door links to a different "room" — but the references are MD5 hashes rather than plain integers, adding a layer of obscurity that doesn't actually provide security.

---

## Reconnaissance

Inspecting the page source revealed an image map where each door's `href` was an MD5 hash:

```html
<area href="c4ca4238a0b923820dcc509a6f75849b" ...>
<area href="c81e728d9d4c2f636f067f89cc14862c" ...>
<area href="eccbc87e4b5ce2fe28308fd9f2a7baf3" ...>
```

### Cracking the Hashes

Using CrackStation, the hashes were quickly identified:

| Hash | Plaintext |
|------|-----------|
| `c4ca4238...` | 1 |
| `c81e728d...` | 2 |
| `eccbc87e...` | 3 |
| `a87ff679...` | 4 |

The pattern is clear — each door corresponds to an integer, hashed with MD5.

---

## Exploitation

### Generating the Missing Hash

The visible doors covered rooms 1–8, but room **0** was not shown. Generating its MD5 hash:

```bash
echo -n "0" | md5sum
# cfcd208495d565ef66e7dff9f98764da
```

### Accessing the Hidden Room

Navigating to:

```
http://<TARGET_IP>/cfcd208495d565ef66e7dff9f98764da
```

Returned the flag.

---

## Key Takeaways

- **Security through obscurity is not security.** Hashing IDs with MD5 doesn't prevent enumeration — it just adds one extra step.
- MD5 is not a secure identifier; it's trivially reversible for known inputs.
- IDOR doesn't require plain numeric IDs — hashed, encoded, or obfuscated references are equally vulnerable if they map predictably to resources.

---

## Vulnerability Summary

| Field | Detail |
|-------|--------|
| Type | IDOR via MD5-hashed identifiers |
| Impact | Access to any room by generating the corresponding hash |
| Fix | Use server-side authorization; never rely on obscured identifiers for access control |
