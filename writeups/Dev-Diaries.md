# Marvenly — TryHackMe Writeup

**Category:** OSINT / Passive Reconnaissance  
**Difficulty:** Easy  
**Objective:** Recover information about a freelance developer who disappeared with the client's source code.

---

## Context

Marvenly hired a freelance developer to build their website. The developer vanished without delivering the source code. The only available information is the primary domain: `marvenly.com`.

---

## Reconnaissance

### Q1 — Development Subdomain

The challenge hints that traces of the development process may still exist online. One of the most effective ways to find subdomains without sending packets to the target is to query public SSL certificate history.

**Tool:** [crt.sh](https://crt.sh)

```
https://crt.sh/?q=marvenly.com
```

**Result:** `uat-testing.marvenly.com`

---

### Q2 — Developer's GitHub Username

With the subdomain in hand, the next step is to search for public repositories related to the project. Freelance developers frequently use GitHub to version their code.

**Search:**
```
https://github.com/search?q=marvenly
```

**Result:** profile `notvibecoder23` with a Marvenly project repository.

---

### Q3 — Developer's Email Address

Git automatically records the author's name and email in every commit. Even if the code was deleted, the commit history remains. By appending `.patch` to any commit URL, the metadata becomes exposed.

**Technique:**
```
https://github.com/notvibecoder23/[repo]/commit/[hash].patch
```

**Result:** `freelancedevbycoder23@gmail.com`

---

### Q4 — Reason for Code Removal

Found directly in the repository's commit history.

**Result:** `The project was marked as abandoned due to a payment dispute`

---

### Q5 — Hidden Flag

Located inside a file in the repository (README or configuration file).

**Flag:** `THM{g1t_h1st0ry_n3v3r_f0rg3ts}`

---

## Key Takeaways

- **crt.sh** is a powerful passive reconnaissance source — SSL certificates are permanent public records.
- Git **never forgets**. Even after deleting files, commit history preserves author metadata (name, email) accessible via `.patch`.
- Freelance developers often link their real identities to client projects without realizing it.

---

## Tools Used

| Tool | Purpose |
|---|---|
| crt.sh | Subdomain enumeration via SSL certificates |
| GitHub Search | Finding project-related repositories |
| Git `.patch` | Extracting commit metadata |
