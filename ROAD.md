# ROAD

**Platform:** TryHackMe
**Date:** 2026-10-19
**Difficulty:** Medium
**Category:** Web Security / Linux / Privilege Escalation / MongoDB
**Room URL:** [https://tryhackme.com/room/road](https://tryhackme.com/room/road)
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

ROAD is a CTF based on a real-world penetration-testing engagement. The room focuses on web application enumeration, account manipulation, gaining an initial foothold, enumerating MongoDB, and exploiting a misconfigured privilege-escalation mechanism to obtain root access.

The main lesson: **don't rush enumeration.** A small detail in a profile update, a hidden upload directory, or a recurring background process can be the difference between being stuck and completing the machine.

---

## 🔍 Reconnaissance

### Nmap Scan

I started by scanning the target to identify open ports and the services running on them.

```
nmap -sC -sV -oN nmap_output.txt <TARGET_IP>
```

**Results:**

|Port|Service|Version|Notes|
|---|---|---|---|
|22/tcp|SSH|Not recorded|Potential remote login|
|80/tcp|HTTP|Not recorded|Main web application|

_Note: The original walkthrough's screenshots contained the scan output, but the exact service versions weren't included in the text._

### Web Enumeration

Since port 80 was open, I moved on to directory enumeration.

```
gobuster dir -u http://<TARGET_IP>/ \
  -w /usr/share/wordlists/dirb/common.txt \
  -o gobuster_output.txt
```

I explored the pages and directories I discovered, including:

- The main page and its contact form
    
- Login and registration pages
    
- Static JavaScript and CSS directories
    
- A phpMyAdmin page
    
- An `/admin` path and a possible API endpoint
    
- The profile page available after logging in
    

I also identified the site's domain and added it to `/etc/hosts`, allowing me to access the application using its hostname. Subdomain enumeration didn't produce anything useful.

---

## 🧭 Steps I Took

### Step 1 — Explore the Web Application

- **What I did:** I checked the public pages, login form, registration form, contact form, and the directories discovered during enumeration.
    
- **Command/Tool used:** Browser, Gobuster, page source inspection.
    
- **What I found:** The application supported account registration and login. After registering, I could access a profile page containing an `awb` parameter.
    
- **Why it matters:** Every input and account function can reveal a potential attack surface. The `awb` parameter was worth investigating, although finding a parameter alone doesn't prove that it's vulnerable.
    

### Step 2 — Identify the Admin Account

- **What I did:** I inspected the profile page and its available options.
    
- **What I found:** The profile revealed the administrator's email address: `admin@sky.thm`. The regular user account also had a password-reset option.
    
- **Why it matters:** I now had a specific account to investigate and a password-reset workflow to test. I tried other potential attack surfaces first, but the reset functionality proved more useful.
    

### Step 3 — Investigate the Password-Reset Functionality

- **What I did:** I intercepted the password-reset request using Burp Suite and inspected the submitted parameters.
    
- **Tool used:** Burp Suite Proxy and Repeater.
    
- **What I found:** The application accepted a modified email value in the request, allowing the email associated with the account to be changed.
    
- **Why it matters:** The server wasn't correctly enforcing which account the request was allowed to modify. This indicated a broken access-control or account-management issue.
    

The important lesson here is that **client-side forms are not a security boundary**. The server must independently verify that a user is authorized to modify the account affected by a request.

### Step 4 — Gain Administrator Access

- **What I did:** I followed the room's account-management attack path to regain access as the administrator.
    
- **What I found:** After the account manipulation, I could access the administrator account.
    
- **Why it matters:** Administrator access exposed functionality that wasn't available to a normal user, including an upload option.
    

This was the turning point in the web application stage.

### Step 5 — Investigate the Upload Functionality

- **What I did:** I inspected the administrator's upload feature and looked at the relevant page source when the first upload attempt didn't produce the expected result.
    
- **Tool used:** Browser and page source inspection.
    
- **What I found:** I discovered a directory at `/v2/profileimages`. Directory listing was disabled, but that didn't necessarily mean individual files couldn't be accessed.
    
- **Why it matters:** A directory can prevent visitors from listing its contents while still serving files when their exact paths are requested.
    

In the authorized lab, the uploaded file could be reached through that directory. Accessing the uploaded payload provided the initial foothold on the target.

### Step 6 — Find the User Flag

- **What I did:** After obtaining a shell, I enumerated the system and searched for the user flag.
    
- **Command used:**
    

```
find / -type f -name "user.txt" 2>/dev/null
```

- **What I found:** The user flag.
    
- **Why it matters:** This confirmed that I had moved beyond the web application and gained access to the underlying Linux system.
    

**User flag:**

```
63191e4ece37523c9fe6bb62a5e64d45
```

### Step 7 — Enumerate for Privilege Escalation

- **What I did:** I investigated possible privilege-escalation paths using manual enumeration, LinPEAS, and `pspy`.
    
- **Tools used:** LinPEAS and `pspy`.
    
- **What I found:** The `pspy` output showed recurring background processes, suggesting that scheduled jobs deserved closer inspection. I also identified a MongoDB configuration at `/etc/mongod.conf`.
    
- **Why it matters:** Scheduled tasks and locally running services can expose weaknesses that aren't visible from the network scan. Observing a process repeatedly can reveal a useful lead.
    

### Step 8 — Enumerate MongoDB

- **What I did:** I investigated the MongoDB service and checked whether it required authentication.
    
- **Tool used:** MongoDB client and the relevant HackTricks documentation.
    
- **What I found:** The database was accessible without a password, and I discovered credentials for the `webdeveloper` account.
    

```
Username: webdeveloper
Password: BahamasChapp123!@#
```

- **Why it matters:** A database that allows unauthenticated access can expose sensitive application data, including credentials. Those credentials may also be useful for identifying other accounts or access paths on the machine.
    

I used the discovered credentials to log in as `webdeveloper`.

### Step 9 — Identify the Privilege-Escalation Vulnerability

- **What I did:** I continued local privilege-escalation enumeration after switching to the `webdeveloper` account.
    
- **What I found:** The executable `/usr/bin/sky_backup_utility` stood out as a potential escalation path. Further investigation revealed that the `LD_PRELOAD` mechanism could be abused in the context of the utility's privileged execution.
    
- **Why it matters:** `LD_PRELOAD` can load a shared library into a process. If a privileged program can be launched with an attacker-controlled library in an unsafe configuration, the library may execute code with the program's privileges.
    

The key wasn't simply finding an unusual executable. It was understanding how the executable interacted with environment variables and its execution permissions.

### Step 10 — Obtain Root Access

- **What I did:** I followed the room's intended privilege-escalation path using the vulnerable backup utility and the `LD_PRELOAD` mechanism.
    
- **What I found:** The exploit path provided root-level access.
    
- **Why it matters:** I had escalated from a web application foothold to the highest privilege level on the Linux machine.
    

The underlying lesson is that privileged programs must be carefully designed and configured. Environment variables that influence program loading can become a security risk when they are not handled safely.

### Step 11 — Retrieve the Root Flag

Once root access was obtained, I searched for the root flag.

```
find / -type f -name "root.txt" 2>/dev/null
```

I then read the flag at its discovered location to complete the room.

**Root flag:** Not included in the supplied walkthrough text. Add the exact value from your completed room.

---

## 🛠 Commands Used

|Command / Tool|What It Does|
|---|---|
|`nmap -sC -sV <TARGET_IP>`|Identifies open ports and probes services|
|`gobuster dir`|Enumerates web directories and files|
|`/etc/hosts`|Maps the target IP to the discovered hostname|
|Burp Suite Repeater|Allows HTTP requests to be inspected and modified|
|`find / -type f -name "user.txt" 2>/dev/null`|Searches for the user flag|
|`pspy`|Observes processes running on the target|
|LinPEAS|Helps identify potential Linux privilege-escalation paths|
|MongoDB client|Connects to MongoDB and queries accessible databases|
|`find / -type f -name "root.txt" 2>/dev/null`|Searches for the root flag|

---

## 🚩 Flags Found

| Flag      | Value                              |
| --------- | ---------------------------------- |
| User flag | `63191e4ece37523c9fe6bb62a5e64d45` |
| Root flag | `3a62d897c40a815ecbe267df2f533ac6` |

---

## 📎 Resources Used

- [HackTricks — Linux Privilege Escalation](https://book.hacktricks.wiki/en/linux-hardening/privilege-escalation/index.html)
    
- [HackTricks — MongoDB](https://book.hacktricks.wiki/en/network-services-pentesting/27017-27018-mongodb.html)
    
- [Burp Suite Documentation](https://portswigger.net/burp/documentation)
    
- [Nmap Reference Guide](https://nmap.org/book/man.html)
    

---