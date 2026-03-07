# Neighbour — TryHackMe Writeup

**Difficulty:** Easy
**Category:** Web
**Vulnerability:** Insecure Direct Object Reference (IDOR)

---

## Overview

Neighbour is a simple web application with a login page. The challenge hints at accessing another user's profile — a textbook IDOR scenario.

---

## Reconnaissance

The login page contained a subtle hint:

> *"Don't have an account? Use the guest account!"*

Logging in with `guest:guest` granted access and redirected to:

```
http://<TARGET_IP>/profile.php?user=guest
```

The `user` parameter directly references the account being viewed.

---

## Exploitation

### Identifying IDOR

The URL structure `?user=guest` suggests the server loads a profile based on the username passed in the parameter, with no authorization check verifying whether the logged-in user has permission to view it.

### Accessing the Admin Profile

Replacing `guest` with `admin` in the URL:

```
http://<TARGET_IP>/profile.php?user=admin
```

The server returned the admin's profile page, which contained the flag.

---

## Key Takeaways

- **IDOR** happens when an application uses user-controlled input to access objects without verifying authorization.
- Even when IDs are usernames instead of numbers, IDOR is still exploitable.
- The fix is server-side: always verify that the authenticated user is authorized to access the requested resource.

---

## Vulnerability Summary

| Field | Detail |
|-------|--------|
| Type | Insecure Direct Object Reference (IDOR) |
| Parameter | `?user=` |
| Impact | Unauthorized access to any user's profile |
| Fix | Enforce server-side authorization checks on every profile request |
