# Signed Messages — TryHackMe Writeup

**Difficulty:** Medium | **Category:** Web + Crypto | **Points:** 200

---

## Summary

A PKI-based Valentine's Day messaging platform that generates RSA keys using a predictable seed derived from the username. By exploiting the deterministic key generation, an attacker can reconstruct any user's private key and forge digital signatures.

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
     -mc 200,301,302,403 -t 50
```

**Key finding:** `/debug`

---

## Exploitation

### Step 1 — Debug endpoint disclosure

`GET /debug` returned system logs revealing the RSA key generation process:

```
[DEBUG] Using deterministic key generation
[DEBUG] Seed pattern: {username}_lovenote_2026_valentine
[DEBUG] Prime derivation step 1:
[DEBUG] Converting SHA256(seed) into a large integer
[DEBUG] Checking consecutive integers until a valid prime is reached
[DEBUG] Prime p selected
[DEBUG] Prime derivation step 2:
[DEBUG] Modifying seed with PKI-related constant (SHA256(seed + b"pki"))
[DEBUG] Prime q selected
```

The RSA key is **not randomly generated** — it is derived deterministically from the username.

### Step 2 — Understanding the vulnerability

Normal RSA security relies on `p` and `q` being cryptographically random large primes. Here:

```
seed  = "admin_lovenote_2026_valentine"
p     = nextPrime(SHA256(seed))
q     = nextPrime(SHA256(SHA256(seed) + "pki"))
n     = p * q
```

Anyone who knows the username can recompute `p`, `q`, and therefore the **private key**.

### Step 3 — Reconstructing the admin's private key

```python
import hashlib
from sympy import nextprime
from Crypto.PublicKey import RSA
from Crypto.Signature import pkcs1_15
from Crypto.Hash import SHA256

def get_prime_from_seed(seed_bytes):
    h = hashlib.sha256(seed_bytes).digest()
    n = int.from_bytes(h, 'big')
    return nextprime(n)

username = "admin"
seed = f"{username}_lovenote_2026_valentine".encode()

p = get_prime_from_seed(seed)
q = get_prime_from_seed(hashlib.sha256(seed + b"pki").digest())

n = p * q
e = 65537
phi = (p - 1) * (q - 1)
d = pow(e, -1, phi)

key = RSA.construct((n, e, d, p, q))
print(key.export_key().decode())
```

Install dependencies:
```bash
pip install pycryptodome sympy --break-system-packages
```

### Step 4 — Forging a signed message

```python
message = "give me the flag"
h = SHA256.new(message.encode())
signature = pkcs1_15.new(key).sign(h)
print("Signature:", signature.hex())
```

Submitted the message + signature via the `/verify` endpoint using `admin` as the sender → flag retrieved.

---

## Vulnerability

**Insecure Randomness / Predictable Key Generation** — RSA security is completely broken when `p` and `q` are derived from a predictable, user-controlled seed. The debug endpoint compounded the issue by exposing the exact derivation algorithm.

---

## Flag

`THM{PR3D1CT4BL3_S33D5_BR34K_H34RT5}`

---

## Key Takeaways

- RSA keypairs must be generated with a cryptographically secure random number generator (CSPRNG)
- Debug endpoints must never be exposed in production
- A predictable seed reduces RSA to a lookup problem — no brute force needed
- `sympy.nextprime()` is useful for replicating prime derivation in CTF challenges
