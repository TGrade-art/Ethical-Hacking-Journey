# RootMe

**Platform:** TryHackMe  
**Date:** 2026-08-22  
**Difficulty:** Easy 🟢  
**Category:** Linux / Web / Privilege Escalation  
**Room URL:** https://tryhackme.com/room/rrootme
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> RootMe was a pretty straightforward Linux privilege-escalation room. The main attack chain was finding a hidden upload panel, bypassing the file-extension filter to upload a PHP reverse shell, getting initial access, and then abusing a misconfigured `sudo` permission to become root.

---

## 🔍 Reconnaissance

### Nmap Scan

I started with Nmap to see what ports and services were running on the machine.

```bash
nmap -sV -sC <MACHINE_IP>
```

The scan showed two interesting open ports:

|Port|Service|Version|Notes|
|---|---|---|---|
|22|SSH|OpenSSH|SSH service|
|80|HTTP|Apache 2.4.41|Web server|

Since HTTP was open, I went straight to the website.

---

### Directory Enumeration

The main webpage didn't give me much, so I moved on to directory enumeration with Dirb.

```bash
dirb http://<MACHINE_IP> /usr/share/dirb/wordlists/common.txt
```

This found an interesting directory:

```text
/panel/
```

That looked much more promising.

---

## 🧭 Steps I Took

### Step 1 — Finding the Upload Panel

- **What I did:** Opened `/panel/` in the browser.
    
- **Command/Tool used:** Browser
    

The page contained a file-upload function.

That immediately got my attention because unrestricted or poorly validated file uploads can sometimes be turned into remote code execution.

My plan was simple:

**Upload a PHP reverse shell → execute it → get a shell.**

---

### Step 2 — Testing the File Extension Filter

I prepared a PHP reverse shell and tried uploading it as:

```text
shell.php
```

The server rejected it with:

```text
PHP não é permitido!
```

So I tried another common PHP extension:

```text
shell.php5
```

That was also blocked.

At this point it was pretty clear that the application was filtering PHP extensions.

So instead of giving up, I looked for an extension that the server would accept while Apache/PHP would still interpret the file as PHP.

---

### Step 3 — Bypassing the Upload Filter

I found an extension/filter bypass that allowed the reverse-shell file to be uploaded.

The server responded with:

```text
arquivo foi upado com sucesso!
```

Translation:

> File uploaded successfully!

Nice. 😏

Now I just needed to trigger the uploaded file while listening for the reverse-shell connection.

---

### Step 4 — Getting the Initial Shell

I started a listener on my machine and accessed the uploaded reverse shell.

Once it connected back, I had command execution on the target.

I checked the `/var/www` directory:

```bash
cd /var/www
ls
```

The user flag was there:

```bash
cat user.txt
```

🚩 **User flag:**

```text
THM{YOU_GOT_A_SH3LL}
```

So initial access was done.

---

## 🔐 Privilege Escalation

### Step 5 — Checking `sudo` Permissions

Now I needed to become root.

I tried:

```bash
sudo -l
```

But there was a problem:

```text
sudo: a terminal is required to read the password; either use the -S option to read from standard input or configure an askpass helper
```

The reverse shell wasn't a proper interactive terminal.

So I needed to upgrade it to a TTY.

---

### Step 6 — Spawning a TTY

I used Python to spawn a proper Bash shell:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Now I had a more interactive shell.

I ran:

```bash
sudo -l
```

again.

This time it worked.

The output showed that my current user could execute several commands with `sudo` without entering a password.

The important part was that dangerous binaries such as:

```text
/usr/bin/sudo
/usr/bin/mount
```

were available with elevated privileges.

That was the privilege-escalation path.

---

### Step 7 — Becoming Root

I abused the allowed `sudo` permissions to execute commands with root privileges.

Eventually I got:

```text
root
```

And just like that, the machine was mine. 💀

---

### Step 8 — Getting the Root Flag

I went to the root user's home directory:

```bash
cd /root
ls
```

Then:

```bash
cat root.txt
```

And got:

```text
THM{PRIVILEGE_ESC4LATION}
```

🚩 **Root flag found.**

---

## 🛠 Commands Used

|Command / Tool|What It Does|
|---|---|
|`nmap -sV -sC <IP>`|Scans ports and identifies services|
|`dirb http://<IP> ...`|Enumerates hidden web directories|
|`python3 -c 'import pty; pty.spawn("/bin/bash")'`|Spawns a pseudo-TTY|
|`sudo -l`|Lists the current user's sudo permissions|
|`cd /var/www`|Moves to the web server directory|
|`cat user.txt`|Reads the user flag|
|`cd /root`|Moves to the root user's home directory|
|`cat root.txt`|Reads the root flag|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|User flag|`THM{YOU_GOT_A_SH3LL}`|
|Root flag|`THM{PRIVILEGE_ESC4LATION}`|

---

## 📎 Resources Used

- TryHackMe — RootMe
    
- Nmap
    
- Dirb
    
- Python PTY
    
- PentestMonkey PHP Reverse Shell
    

---
