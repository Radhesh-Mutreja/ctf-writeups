# TryHackMe — CyberHeroes CTF

## Overview

CyberHeroes is a simple web-based CTF focused on finding a way around a login page.

The interesting part is that the application performs the authentication **client-side**, meaning the credentials can be found by inspecting the website's source code and JavaScript.

---

## Enumeration

I started with an Nmap scan against the target:

```bash
nmap -sC -sV <MACHINE_IP>
```

The scan showed two relevant services:

```text
22/tcp   open   ssh
80/tcp   open   http
```

Port **80** was running a web server, so I opened the target in a browser.

---

## Web Enumeration

The website contained several sections, including a login page.

Navigating to:

```text
http://<MACHINE_IP>/login.html
```

presented a username and password form.

I first tried some basic credentials, but the login failed.

At this point, rather than trying to brute-force the login, I checked how the application actually handled authentication.

---

## Inspecting the Login

I opened Burp Suite and attempted another login while interception was enabled.

Interestingly, there was **no request being sent to the server** when the credentials were submitted.

That suggested the authentication logic was happening directly inside the browser.

This is a pretty big clue.

If the server isn't receiving the credentials, the JavaScript running on the page must be responsible for checking them.

---

## Finding the Credentials

I inspected the page source and found the JavaScript responsible for authentication.

The relevant logic was essentially:

```javascript
a = document.getElementById('uname')
b = document.getElementById('pass')

if (a.value == "h3ck3rBoi" &&
    b.value == ReverseString("54321@terceSrepuS"))
```

This gave us the username immediately:

```text
h3ck3rBoi
```

The password was slightly more interesting.

The application was reversing this string:

```text
54321@terceSrepuS
```

So I reversed it.

Using Linux:

```bash
echo '54321@terceSrepuS' | rev
```

Result:

```text
SuperSecret@12345
```

Therefore the credentials were:

```text
Username: h3ck3rBoi
Password: SuperSecret@12345
```

---

## Logging In

I entered the credentials into the login form:

```text
Username: h3ck3rBoi
Password: SuperSecret@12345
```

The authentication succeeded and the flag was revealed.

```text
flag{edb0be532c540b1a150c3a7e85d2466e}
```

---

## Attack Path

```text
Nmap
  |
  v
Port 80
  |
  v
Web Login
  |
  v
No server-side authentication request
  |
  v
Inspect JavaScript
  |
  v
Hardcoded username + reversed password
  |
  v
Reverse password
  |
  v
Login
  |
  v
Flag
```

---

## Key Takeaways

### Don't immediately brute-force a login

Before trying large wordlists, understand how the authentication mechanism works.

Here, there wasn't even a server-side authentication request to attack.

### Client-side authentication is insecure

Credentials should **never** be stored or validated entirely in client-side JavaScript.

Anyone who can access the page can inspect the JavaScript and recover the logic.

### Source code is worth checking

For web CTFs, always look at:

* Page source
* JavaScript
* HTML comments
* API requests
* Hidden endpoints
* Client-side validation

Sometimes the vulnerability is sitting in plain sight.

---

## Tools Used

```text
Nmap
Burp Suite
Browser Developer Tools
Linux
```

## Flag

```text
flag{edb0be532c540b1a150c3a7e85d2466e}
```

## Conclusion

CyberHeroes was a straightforward introduction to **client-side authentication flaws**.

The important lesson wasn't really the password itself—it was recognizing that the login was being validated in the browser rather than by the server.

Once that was identified, inspecting the JavaScript exposed the credentials and made the rest of the challenge pretty much a one-command password reversal.
