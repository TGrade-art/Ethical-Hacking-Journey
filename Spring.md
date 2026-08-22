# Spring

**Platform:** TryHackMe  
**Date:** 2026-08-21
**Difficulty:** Hard 
**Category:** Web / Linux / Git / Privilege Escalation  
**Room URL:** https://tryhackme.com/room/spring
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> This was a proper blind challenge. No obvious hints, just me and the box. 😭 The attack chain ended up going from a clean web page → exposed Git repository → leaked Spring configuration → Actuator abuse → RCE → password reuse → root through a systemd/logging misconfiguration. This one had some really nice rabbit holes.

---

## 🔍 Reconnaissance

### Nmap Scan

First things first — **Nmap**. :)

```bash
nmap -sV -p- <TARGET_IP>
```

**Results:**

|Port|Service|Version|Notes|
|---|---|---|---|
|22|SSH|OpenSSH 7.6p1 Ubuntu 4ubuntu0.3|SSH|
|80|HTTP|—|Web server|
|443|HTTPS|—|HTTPS web server|

HTTPS immediately caught my attention, so I opened it in the browser.

And...

> **Hello, World!**

That's it. 😂

Way too clean.

I checked the response with:

```bash
curl https://<TARGET_IP>/ -k -v
```

The headers and certificate gave me a hostname of `spring.thm`, so I added it to `/etc/hosts`.

Still got the same **Hello, World!** page.

So I moved on to enumeration.

---

## 🧭 Steps I Took

### Step 1 — Finding the Exposed Git Repository

- **What I did:** Ran directory/file enumeration against the web server and focused on the `/sources/` directory.
    
- **Command/Tool used:**
    

```bash
dirsearch -u https://<TARGET_IP>/sources/ -t 200 -E -r -R 5
```

- **What I found:** A whole bunch of `.git` files returning `200`.
    

Some of the interesting ones were:

```text
.git/config
.git/HEAD
.git/index
```

That was a **very** good sign. 

An exposed `.git` directory can potentially give us the application's source code _and its entire commit history_.

So obviously, I wanted the repo.

---

### Step 2 — Dumping the Git Repository

- **What I did:** Used GitTools' `gitdumper.sh` to download the exposed repository.
    
- **Command/Tool used:**
    

```bash
./gitdumper.sh https://<TARGET_IP>/sources/new/.git
```

- **What I found:** The Git repository and its history.
    
- **Why it matters:** Git history can contain old files, credentials, configuration, and other information that isn't necessarily present in the current version of the application.
    

I checked the commit history:

```bash
git log
```

One commit immediately stood out:

```text
commit 1a83ec34...
Author: John Smith <johnsmith@spring.thm>

added greeting
changed security password to my usual format
```

**"changed security password to my usual format"**

👀

Yeah... I'm definitely checking that commit.

I reset to it and found:

```text
application.properties
```

And this is where things started getting interesting.

---

### Step 3 — Analyzing `application.properties`

- **What I did:** Examined the old Spring configuration file.
    
- **Command/Tool used:** Git / text editor
    
- **What I found:** Several useful pieces of information:
    

```text
x-9ad42dea0356cb04
Spring Actuator configuration
Spring Security credentials
H2 database configuration
```

The custom header was especially interesting.

I tried accessing the Actuator endpoint normally:

```text
/actuator/
```

But got:

```text
403 Forbidden
```

So the endpoint existed, but there was clearly another check happening.

The custom header from the configuration looked like a possible way around it.

---

### Step 4 — Getting Access to Spring Actuator

- **What I did:** Sent the custom header with the Actuator request.
    
- **Command/Tool used:**
    

```bash
curl https://<TARGET_IP>/actuator/ \
-H 'x-9ad42dea0356cb04: 172.16.0.21' \
-k
```

- **What I found:** The Actuator endpoints were now accessible.
    

BOOM. 😏

The Actuator menu exposed a bunch of functionality that shouldn't have been available to an attacker.

---

### Step 5 — Getting a Shell as `nobody`

- **What I did:** Investigated the exposed Actuator functionality and the application's H2 database configuration.
    
- **What I found:** I was able to use the H2 functionality to reach command execution.
    

I tested the command execution with a simple ping first to make sure it actually worked.

Once that worked, I went for a reverse shell.

Eventually:

```bash
whoami
```

returned:

```text
nobody
```

So I had my foothold.

---

### Step 6 — Getting the First Flag

With the `nobody` shell, I searched for the foothold flag:

```bash
find / -name foothold.txt 2>/dev/null
```

Then:

```bash
cat /opt/foothold.txt
```

🚩 **Foothold flag:**

```text
THM{dont_expose_.git_to_internet}
```

Pretty fitting considering the entire attack chain started with the exposed Git repository. 😂

---

### Step 7 — Moving From `nobody` to `johnsmith`

- **What I did:** Investigated the available users and how SSH was configured.
    
- **What I found:** SSH was enabled, but it was configured for public-key authentication.
    

So SSH wasn't immediately useful.

I went back to the Git history and remembered the commit message:

```text
changed security password to my usual format
```

That strongly suggested password reuse or a predictable password format.

I generated a wordlist based on the information I had and brute-forced the password for `johnsmith`.

12 hours later...

### Credentials Found

```text
johnsmith:PrettyS3cureAccountPassword123
```

😂 **"Pretty Secure"** indeed.

I switched users:

```bash
su johnsmith
```

Then grabbed the user flag:

```bash
cat user.txt
```

🚩 **User flag:**

```text
THM{this_is_still_password_reuse}
```

---

### Step 8 — Finding the Root Escalation Path

Now I had access to `johnsmith`, so it was time to start looking for a way to root the machine.

I found an interesting systemd service:

```bash
cat /etc/systemd/system/spring.service
```

The service was running as **root** and was writing application logs using `tee` into:

```text
/home/johnsmith/tomcatlogs/
```

The log filenames were based on epoch timestamps.

That's when the symlink idea clicked.

If the logging process follows a symlink and runs as root, I may be able to make it write somewhere I shouldn't normally have access to.

---

### Step 9 — Getting Root

The attack chain was basically:

1. Stop the application using the exposed Actuator functionality.
    
2. Create a symlink targeting:
    

```text
/root/.ssh/authorized_keys
```

3. Use the application's **Hello World** endpoint to inject my SSH public key into the log output.
    
4. The root process follows the symlink and writes my key into root's `authorized_keys`.
    
5. SSH into the machine as root using my private key.
    

Then:

```bash
ssh -i key root@<TARGET_IP>
```

And...

```text
root
```

**WE ARE ROOT.** :))

---

### Step 10 — Getting the Root Flag

Finally:

```bash
cat root.txt
```

🚩 **Root flag:**

```text
THM{sshd_does_not_mind_the_junk}
```

And that's the box.

---

## 🛠 Commands Used

|Command / Tool|What It Does|
|---|---|
|`nmap -sV -p- <IP>`|Scans all TCP ports and identifies services|
|`curl ... -k -v`|Inspects HTTPS responses and certificates|
|`dirsearch`|Enumerates files and directories|
|`git log`|Examines Git commit history|
|`gitdumper.sh`|Downloads an exposed Git repository|
|`curl ... -H 'header: value'`|Sends a custom HTTP header|
|`find / -name foothold.txt 2>/dev/null`|Searches the filesystem for the foothold flag|
|`su johnsmith`|Switches to the `johnsmith` account|
|`cat /etc/systemd/system/spring.service`|Examines the Spring systemd service|
|`ssh -i key root@<IP>`|Connects to the machine as root using an SSH key|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|Foothold flag|`THM{dont_expose_.git_to_internet}`|
|User flag|`THM{this_is_still_password_reuse}`|
|Root flag|`THM{sshd_does_not_mind_the_junk}`|

---

## 💡 What I Learned

- **Never expose `.git` directories to the internet.** The repository can contain way more information than the current website, including old configuration files and credentials.
    
- **Git history is part of enumeration.** Finding a repository isn't the end — old commits can contain secrets that developers thought they removed.
    
- **Spring Actuator needs to be secured properly.** An exposed management endpoint can give attackers a ridiculous amount of information and functionality.
    
- **Password reuse is still a massive problem.** The Git commit literally hinted that the developer used their "usual" password format, which ended up being enough to move from `nobody` to `johnsmith`.
    
- **Systemd services running as root deserve serious attention.** The logging configuration created a path from a low-privileged user to writing into a root-owned SSH configuration.
    
- **The best part of this room was following the clues.** The `.git` leak led to the configuration, the configuration led to Actuator, Actuator led to `nobody`, Git history led to `johnsmith`, and the systemd service finally led to root.
    

---

## ❓ What Confused Me / What to Research Next

- I want to understand **Spring Actuator security** better, especially how exposed endpoints can lead to command execution.
    
- I want to research **H2 database injection and RCE** more because that was probably the most interesting part of the initial foothold.
    
- I also want to learn more about **symlink attacks against root processes**, especially how they can be used against logging mechanisms.
    
- The password-generation part was also interesting, so I'd like to get better at creating targeted wordlists from information gathered during enumeration.
    

---

## 🔗 Linked Notes

- [Nmap](https://chatgpt.com/c/Nmap)
    
- [Git Enumeration](https://chatgpt.com/c/Git%20Enumeration)
    
- [GitTools](https://chatgpt.com/c/GitTools)
    
- [Spring Actuator](https://chatgpt.com/c/Spring%20Actuator)
    
- [H2 Database](https://chatgpt.com/c/H2%20Database)
    
- [Reverse Shells](https://chatgpt.com/c/Reverse%20Shells)
    
- [Password Reuse](https://chatgpt.com/c/Password%20Reuse)
    
- [Linux Privilege Escalation](https://chatgpt.com/c/Linux%20Privilege%20Escalation)
    
- [Systemd](https://chatgpt.com/c/Systemd)
    
- [Symlink Attacks](https://chatgpt.com/c/Symlink%20Attacks)
    
- [SSH](https://chatgpt.com/c/SSH)
    

---

## 📎 Resources Used

- TryHackMe — Spring
    
- GitTools
    
- Nmap
    
- Spring Actuator documentation
    
- H2 Database documentation
    

---

_Report written in_ [_TryHackMe Vault_](https://chatgpt.com/c/TryHackMe%20Vault) _— part of my ethical hacking journey 🛡️_