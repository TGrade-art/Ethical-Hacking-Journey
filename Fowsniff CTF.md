# Fowsniff CTF — TryHackMe Write-Up

**Room:** [Fowsniff CTF](https://tryhackme.com/room/fowsniff)  
**Difficulty:** Easy  
**Focus:** Enumeration, OSINT, password cracking, POP3, SSH, Linux privilege escalation, reverse shells

## Overview

The **Fowsniff CTF** room is a beginner-friendly penetration-testing challenge built around a fictional company whose email infrastructure has been compromised.

The attack chain is fairly realistic:

**Network enumeration → OSINT → leaked password hashes → password recovery → POP3 access → leaked SSH credentials → SSH access → local enumeration → writable SSH banner → privilege escalation**

The interesting part of the room is that no single vulnerability gives complete access. Instead, information gathered at each stage becomes useful later in the attack.

---

# Task 1 — Deploy the Machine

Start the Fowsniff machine and connect to it through either the **TryHackMe AttackBox** or your own machine using the TryHackMe VPN.

If you're using OpenVPN, it provides the network connection required to communicate with the target machine from your own system.

**Answer:** No answer needed.

---

# Task 2 — Network Enumeration

The first step is determining what services the target exposes.

A comprehensive Nmap scan can be performed with:

```bash
nmap -A -p- -sV <MACHINE_IP>
```

### What the options do

- `-p-` — scans all 65,535 TCP ports.
    
- `-sV` — attempts to identify service versions.
    
- `-A` — enables several advanced enumeration features, including default scripts and OS/service detection.
    

A quicker initial scan can also be useful:

```bash
nmap -sC -sV <MACHINE_IP>
```

The important open ports are:

|Port|Service|Significance|
|---|---|---|
|22|SSH|Remote shell access|
|80|HTTP|Web server|
|110|POP3|Email retrieval|
|143|IMAP|Email access|

The presence of **POP3 and IMAP** is particularly interesting because the scenario revolves around compromised company email accounts.

**Answer:** No answer needed.

---

# Task 3 — Investigating the Services

Let's break down the exposed services.

### Port 22 — SSH

**SSH (Secure Shell)** provides encrypted remote access to a machine.

If valid credentials are discovered later, this could provide a direct command-line session on the target.

### Port 80 — HTTP

The HTTP service hosts the company's website.

Visiting:

```text
http://<MACHINE_IP>
```

reveals the Fowsniff website.

The website is also useful for gathering information about the fictional company and its employees.

### Port 110 — POP3

**POP3 (Post Office Protocol version 3)** is an email retrieval protocol.

An exposed POP3 service suggests that users may be able to authenticate and retrieve their mail remotely.

### Port 143 — IMAP

**IMAP (Internet Message Access Protocol)** is another email protocol.

Unlike traditional POP3 usage, IMAP is designed around keeping messages on the mail server and synchronizing them with clients.

For this room, however, the POP3 service becomes especially useful later.

**Answer:** No answer needed.

---

# Task 4 — OSINT: Finding Public Information

The room gives an important clue:

> The attackers were able to hijack Fowsniff Corporation's official Twitter account.

This means the company's public social-media presence becomes part of our attack surface.

Looking through the company's public information eventually leads to a **Pastebin dump containing password hashes**.

The leaked credentials are represented by MD5 hashes.

This is a classic example of why **OSINT (Open-Source Intelligence)** matters during penetration testing: information exposed publicly can become just as useful as a technical vulnerability.

**Answer:** No answer needed.

---

# Task 5 — Recovering the MD5 Passwords

The leaked passwords are stored as MD5 hashes.

MD5 is a cryptographic hashing algorithm, but it is **not considered suitable for securely storing passwords today**.

A hash is designed to be one-way, meaning you don't simply "decrypt" an MD5 hash. Instead, an attacker can try candidate passwords, hash them, and compare the results against the stolen hash.

The recovered credentials are:

|Username|MD5 Hash|Password|
|---|---|---|
|`mauer`|`8a28a94a588a95b80163709ab4313aa4`|`mailcall`|
|`mustikka`|`ae1644dac5b77c0cf51e0d26ad6d7e56`|`bilbo101`|
|`tegel`|`1dc352435fecca338acfd4be10984009`|`apples01`|
|`baksteen`|`19f5af754c31f1e2651edde9250d69bb`|`skyler22`|
|`seina`|`90dc16d47114aa13671c697fd506cf26`|`scoobydoo2`|
|`stone`|`a92b8a29ef1183192e3d35187e0cfabd`|Unknown|
|`mursten`|`0e9588cb62f4b6f27e33d449e2ba0b3b`|`carp4ever`|
|`parede`|`4d6e42f56e127803285a0a7649b5ab11`|`orlando12`|
|`sciana`|`f7fd98d380735e859f8b2ffbbede5a7e`|`07011972`|

**Answer:** No answer needed.

---

# Task 6 — Brute-Forcing the POP3 Login

We now have a list of usernames and recovered passwords.

The next question is whether any of these credentials work against the exposed POP3 service.

Metasploit contains a POP3 login scanner that can test a username/password list against the service.

Start Metasploit:

```bash
msfconsole
```

Search for POP3 login modules:

```text
search pop3 login
```

Select the appropriate POP3 login module and inspect its configuration:

```text
show options
```

Set the target and credential files:

```text
set RHOSTS <MACHINE_IP>
set USER_FILE users.txt
set PASS_FILE passwd.txt
set VERBOSE false
```

Then run the module:

```text
run
```

The successful credentials reveal access to one of the employee email accounts.

The important account is:

```text
Username: seina
Password: scoobydoo2
```

**Answer:** No answer needed.

---

# Task 7 — Seina's Email Password

The recovered password for Seina's email account is:

```text
scoobydoo2
```

**Answer:**

> `scoobydoo2`

---

# Task 8 — Accessing the POP3 Mailbox

Now that we have valid POP3 credentials, we can connect directly to the mail service.

For the room's environment, a simple TCP connection can be made with:

```bash
nc <MACHINE_IP> 110
```

Authenticate:

```text
USER seina
PASS scoobydoo2
```

If authentication succeeds, the server provides access to the mailbox.

We can inspect the available messages using:

```text
LIST
```

This reveals the messages stored in Seina's mailbox.

The important point here is that **email itself has become a credential-disclosure mechanism**.

**Answer:** No answer needed.

---

# Task 9 — Finding the Temporary SSH Password

Retrieve the messages using:

```text
RETR 1
```

and:

```text
RETR 2
```

One of the messages contains a temporary password for SSH access:

```text
S1ck3nBluff+secureshell
```

This is the crucial transition from **email access** to **system access**.

**Answer:**

> `S1ck3nBluff+secureshell`

---

# Task 10 — SSH Access

We now have credentials that can be used against the SSH service.

Connect with:

```bash
ssh <username>@<MACHINE_IP>
```

After logging in, verify the current account:

```bash
id
```

The account is a regular user rather than root.

That means we need to perform local enumeration and find a way to escalate our privileges.

---

# Local Enumeration

A useful clue from the room is to investigate files belonging to the `users` group.

Run:

```bash
find / -group users -type f 2>/dev/null
```

Among the results is:

```text
/opt/cube/cube.sh
```

Inspect the file:

```bash
cat /opt/cube/cube.sh
```

The script is associated with the SSH login banner.

This is important because a script involved in the SSH login process can potentially provide code execution when it is modified and subsequently executed.

---

# Privilege Escalation Through `cube.sh`

The key issue is that the logged-in user has permissions that allow the script to be modified.

That creates an opportunity to insert commands that execute when the SSH banner script runs.

For the TryHackMe lab, the room provides a reverse-shell payload that can be adapted to the attacker's IP and listening port.

Set up a listener on the attacker machine:

```bash
nc -lvnp <PORT>
```

Then modify the vulnerable script with the room's supplied reverse-shell command, replacing the IP address and port with the values appropriate to your lab connection.

Once the script executes during a subsequent SSH connection, the callback reaches the listener.

At that point, the privilege-escalation portion of the room is complete.

**Answer:** No answer needed.

---

# Attack Chain Summary

The entire Fowsniff attack can be summarized as:

```text
Nmap
  │
  ├── 22 SSH
  ├── 80 HTTP
  ├── 110 POP3
  └── 143 IMAP
        │
        ▼
      OSINT
        │
        ▼
   Public password dump
        │
        ▼
     MD5 hashes
        │
        ▼
   Password recovery
        │
        ▼
   POP3 authentication
        │
        ▼
      Seina's email
        │
        ▼
 Temporary SSH password
        │
        ▼
      SSH access
        │
        ▼
   Local enumeration
        │
        ▼
    cube.sh discovered
        │
        ▼
  Script modification
        │
        ▼
   Privilege escalation
```

## Key Lessons

### 1. Enumeration comes first

The four exposed services immediately give us several possible attack surfaces. Nmap isn't merely about finding "open ports"; it helps determine **where the investigation should go next**.

### 2. OSINT can become an initial-access vector

The technical attack didn't begin with exploiting the web server. Publicly exposed company information eventually led to credential material.

### 3. Password hashing matters

MD5 is unsuitable for modern password storage. Fast hashing algorithms such as MD5 make password-guessing attacks much more practical when hashes are leaked.

### 4. Credentials are often reused across services

A password discovered through one service becomes much more valuable if it also works against another service such as POP3 or SSH.

### 5. Email can expose infrastructure credentials

The POP3 mailbox contained a temporary SSH password. This demonstrates why sensitive credentials should never be sent through ordinary email without appropriate protections.

### 6. Local enumeration is essential after gaining a shell

Getting a normal user shell doesn't mean the assessment is over. Examining permissions, groups, scripts, scheduled tasks, services, and unusual files can reveal privilege-escalation paths.

---

# Final Answers

|Task|Answer|
|---|---|
|Task 1|No answer needed|
|Task 2|No answer needed|
|Task 3|No answer needed|
|Task 4|No answer needed|
|Task 5|No answer needed|
|Task 6|No answer needed|
|**Task 7**|**`scoobydoo2`**|
|Task 8|No answer needed|
|**Task 9**|**`S1ck3nBluff+secureshell`**|
|Task 10|No answer needed|

This version keeps the actual **Fowsniff** attack path intact while making the reasoning between each stage much clearer, rather than just listing commands.