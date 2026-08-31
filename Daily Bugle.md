# Daily Bugle

**Platform:** TryHackMe  
**Room:** Daily Bugle  
**Difficulty:** Hard
**Category:** Web / SQL Injection / Privilege Escalation  
**Room URL:** [https://tryhackme.com/room/dailybugle](https://tryhackme.com/room/dailybugle)  
**Status:** ✅ Completed

---

## 🧭 Introduction

The goal of the **Daily Bugle** room is to compromise a vulnerable Joomla CMS, obtain administrator access through SQL injection, crack a leaked password hash, and then escalate privileges to root by abusing `yum`.

This room covers several important techniques:

- Web enumeration 🔎
    
- Joomla vulnerability research
    
- SQL injection
    
- Password hash cracking
    
- CMS exploitation
    
- Linux enumeration
    
- Privilege escalation using `yum`
    

Let's get started. 👇

---

# 🔍 1. Reconnaissance

I started with a port scan to see what services were running on the target machine.

```bash
nmap -sV -p- 10.10.213.73
```

The scan revealed an HTTP service, so the web server immediately became one of my main targets.

I opened the IP address in my browser and found the **Daily Bugle** website.

The page also contained a question:

> **Who robbed the bank?**

The answer was:

```text
Spiderman
```

💭 The homepage itself wasn't giving me much useful information for gaining access, so I moved on to enumerating the web application.

---

# 🌐 2. Web Enumeration

While exploring the website, I discovered an `/administrator` directory.

Opening it brought me to a **Joomla administrator login page**.

That immediately made me want to identify the Joomla version because an outdated CMS version can sometimes have publicly known vulnerabilities.

After checking the site information, I found that the Joomla version was:

```text
3.7.0
```

I searched for vulnerabilities affecting Joomla 3.7.0 and found a known **SQL injection vulnerability**.

💭 This was a really important discovery because instead of trying to brute-force the administrator login, I could investigate the SQLi vulnerability and see what information I could extract from the database.

---

# 💉 3. Testing the SQL Injection

The vulnerable parameter was:

```text
list[fullordering]
```

I first tested the parameter manually using a time-based SQL injection payload.

```text
http://10.10.213.73/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=(SELECT * FROM (SELECT(SLEEP(5)))GDiu)
```

The response delay indicated that the parameter was vulnerable to SQL injection.

I then used `sqlmap` to automate the database enumeration.

For example:

```bash
sqlmap -u "http://10.10.213.73/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering]
```

This allowed me to investigate the database and confirm that the SQL injection could be exploited.

💭 At this point I knew I had a way into the Joomla database. Now I needed to figure out what useful information I could extract from it.

---

# 🐍 4. Exploiting Joomla and Getting the Hash

Instead of continuing everything manually, I found a public proof-of-concept exploit for the Joomla vulnerability.

I downloaded it using:

```bash
wget https://raw.githubusercontent.com/stefanlucas/Exploit-Joomla/master/joomblah.py
```

I then ran the exploit against the target.

The exploit revealed credentials for a Joomla account named:

```text
jonah
```

The password wasn't given in plaintext. Instead, I obtained a password hash.

After identifying the hash format, I determined that it was **bcrypt**.

💭 Getting a hash is only half the job. Since bcrypt is designed to be slow and resistant to brute-force attacks, I needed to use a password-cracking tool rather than simply decoding it.

---

# 🔓 5. Cracking the Password

I used **John the Ripper** to attempt to crack the bcrypt hash.

Eventually, John recovered the password:

```text
spiderman123
```

So I now had:

```text
Username: jonah
Password: spiderman123
```

The next step was to try these credentials against the Joomla administrator panel.

---

# 👑 6. Joomla Administrator Access

I logged into:

```text
/administrator
```

using the `jonah` credentials.

It worked! 🎉

The account had **Super User** privileges.

That was a major breakthrough because administrator access meant I could modify parts of the Joomla installation.

I navigated to the template section and found that I could edit the frontend template's `index.php`.

I modified the PHP file to include a reverse-shell payload, making sure to use my own lab listener IP and port.

I started my listener and then triggered the modified page.

This gave me a shell on the target machine.

💭 This is a good example of why CMS administrator access can be extremely powerful. If an administrator can modify PHP templates, they may effectively be able to execute code on the underlying server.

---

# 🕵️ 7. Local Enumeration

Once I had access to the server, I started looking around for useful information.

The Joomla installation was located under:

```text
/var/www/html
```

I inspected the files there and found:

```text
configuration.php
```

This file was especially interesting because Joomla's configuration file can contain database credentials and other sensitive information.

While reading it, I discovered credentials that appeared to belong to another user.

I initially tried using the discovered password for the `root` account, but that didn't work.

Instead, the credentials worked for:

```text
jjameson
```

So I switched to the `jjameson` account.

---

# 🚩 8. Getting the User Flag

After logging in as `jjameson`, I searched the user's files for anything interesting.

Eventually, I found the first flag.

```text
user.txt
```

The flag was:

```text
[USER FLAG]
```

🏁 **User flag captured.**

💭 At this point I had completed the initial compromise and obtained the user-level access. The remaining goal was privilege escalation.

---

# ⬆️ 9. Privilege Escalation

I started by checking what commands I could execute with `sudo`.

My initial attempts didn't immediately give me a way to become root, so I continued enumerating the system.

During my research, I discovered that **`yum` can be abused for privilege escalation when it is allowed through sudo with elevated privileges**.

I checked whether the current user had the necessary permissions to execute it.

Once I confirmed that `yum` was available through the required sudo permissions, I followed the known escalation technique.

This allowed me to execute commands with root privileges.

Eventually, I obtained a root shell.

```text
root
```

🔥 Root access achieved!

---

# 🚩 10. Getting the Root Flag

With root access, I checked the root user's files and found:

```text
/root/root.txt
```

Reading the file gave me the final flag.

```text
[ROOT FLAG]
```

🏆 **Root flag captured!**

---

# 🛠️ Commands Used

|Command|What it does|
|---|---|
|`nmap -sV -p- <IP>`|Scans all TCP ports and identifies services|
|`wget <URL>`|Downloads the Joomla exploit|
|`sqlmap`|Automates SQL injection testing and database enumeration|
|`john`|Attempts to crack password hashes|
|`cat configuration.php`|Reads the Joomla configuration file|
|`sudo -l`|Shows commands the current user can run with sudo|
|`yum`|Package manager that can be abused for privilege escalation when improperly permitted|
|`cat user.txt`|Displays the user flag|
|`cat /root/root.txt`|Displays the root flag|

---

# 💡 What I Learned

- **Joomla version enumeration is important.** Finding the exact CMS version allowed me to search for vulnerabilities that applied specifically to Joomla 3.7.0.
    
- **SQL injection can expose sensitive database information.** In this case, the vulnerability eventually allowed me to obtain a password hash.
    
- **Password hashes aren't automatically useless.** Even though the password wasn't stored in plaintext, weak passwords can still be recovered through password cracking.
    
- **CMS administrator access can lead to code execution.** Being able to modify PHP templates gave me a path from Joomla access to a shell on the underlying machine.
    
- **Configuration files are worth checking.** `configuration.php` contained credentials that helped me move from the web server to another local user.
    
- **`sudo -l` is one of the first commands I check after getting a shell.** Misconfigured sudo permissions can provide a direct route to privilege escalation.
    
- **Small permissions mistakes can completely compromise a machine.** Allowing a powerful program such as `yum` to run with elevated privileges can turn a low-privileged account into root.
    

---

# ❓ What Confused Me / What I Want to Research Next

- I want to understand **exactly how the Joomla 3.7.0 SQL injection works internally**, instead of relying mainly on the public exploit.
    
- I want to learn more about **how bcrypt hashes are structured** and why bcrypt is significantly harder to crack than older hashing algorithms.
    
- I want to understand **why the `yum` privilege escalation works internally**, rather than simply following the known technique.
    
- I also want to get better at **manual SQL injection**, so I can understand what `sqlmap` is doing instead of depending on automation.
    

---

# 🔗 Linked Notes

- [[SQL Injection]]
    
- [[Joomla]]
    
- [[Web Enumeration]]
    
- [[Password Cracking]]
    
- [[John the Ripper]]
    
- [[Linux Privilege Escalation]]
    
- [[Sudo]]
    
- [[YUM]]
    
- [[Reverse Shells]]
    

---

# 📚 Conclusion

The **Daily Bugle** room was a really good chain of different techniques.

I started with basic reconnaissance, found an outdated Joomla installation, discovered the SQL injection vulnerability, extracted a password hash, cracked it, and used the credentials to gain administrator access.

From there, I modified the Joomla template to get a shell, found another user's credentials in `configuration.php`, and finally escalated to root through a `yum` misconfiguration.

The biggest thing I took away from this room is that an attack doesn't always depend on one huge vulnerability. Sometimes it's a chain:

**Recon → Vulnerable Joomla → SQLi → Hash → Cracked Password → Admin Access → Shell → Credentials → Privilege Escalation → Root** 🕷️🔥

Another room completed. ✅