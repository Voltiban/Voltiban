# Sneaky Viper — TryHackMe Writeup

**Category:** OSINT / Threat Intelligence  
**Difficulty:** Easy-Medium  
**Objective:** Track down the three operators of the "Sneaky Viper" hacker group starting from a single handle found on a forum.

---

## Context

The company TryTelecomMe discovered their database was being sold online by the **Sneaky Viper** group. The only available data is the first operator's handle: `@v3n0mbyt3_`.

---

## Task 1 — Operator 1: v3n0mbyt3_

### Finding the alternative platform

**Tool:** Sherlock — searches for a username across multiple platforms simultaneously.

```bash
sherlock v3n0mbyt3_
```

Among the results, the **Threads** profile stood out with real activity.

**Answer Q1:** `threads`

### Flag — Base64 hidden in a reply

Browsing the Threads profile replies, a second user (`_myst1cv1x3n_`) left a suspicious string in the comments:

```
VEhNe3NsMXRoM3J5X3R3MzN0el80bmRfbDM0a3lfcjNwbDEzcyF9
```

Decoding via Base64:

```bash
echo "VEhNe3NsMXRoM3J5X3R3MzN0el80bmRfbDM0a3lfcjNwbDEzcyF9" | base64 -d
```

**Flag:** `THM{sl1th3ry_tw33tz_4nd_l34ky_r3pl13s!}`

---

## Task 2 — Operator 2: _myst1cv1x3n_

### Identification

The handle `_myst1cv1x3n_` was discovered in the Threads reply from the previous task.

**Answer Q1:** `_myst1cv1x3n_`

### Alternative platform and flag

The Threads profile linked directly to **Instagram**. On Instagram, one of the posts linked to **SoundCloud**. In the description of one of the tracks, a Base64 string was found:

```bash
echo "[string]" | base64 -d
```

**Flag:** `THM{s0cm1nt_00ps3c_f1ng3r_m1scl1ck}`

---

## Task 3 — Operator 3: sh4d0wF4NG

### Identification

By tracking interactions between the previous profiles (likes, comments, follows on SoundCloud), the third operator was identified.

**Answer Q1:** `sh4d0wF4NG`

### Alternative platform

```bash
sherlock sh4d0wF4NG
```

Active profile on **GitHub** with three relevant repositories: `red-team-infra` (Terraform/HCL), `evilginx2` (forked phishing framework), and `gophish` (phishing toolkit).

**Answer Q2:** `github`

### Flag — Commit history

Inside the `red-team-infra` repository, the commit history contained credentials and an embedded flag.

**Flag:** `THM{sh4rp_f4ngz_l34k3d_bl00dy_pw}`

---

## Key Takeaways

- **Poor OpSec** is the biggest enemy of threat actors: linking accounts across platforms creates a traceable chain.
- **Sherlock** automates username searches across dozens of platforms — an essential OSINT tool for person-based investigations.
- **Base64** is frequently used to obfuscate sensitive data in public posts — always check seemingly random strings.
- The full chain in this lab: `Threads → Instagram → SoundCloud → Base64 → Flag` — OSINT rarely ends at the first platform.

---

## Tools Used

| Tool | Purpose |
|---|---|
| Sherlock | Username search across multiple platforms |
| Base64 (terminal) | Decoding obfuscated messages |
| GitHub | Repository and commit history analysis |

```bash
# Install Sherlock
pip install sherlock-project --break-system-packages

# Encode/Decode Base64
echo "text" | base64
echo "dGV4dA==" | base64 -d
```
