# ContainMe

**Room:** [TryHackMe — ContainMe](https://tryhackme.com/room/containme1?utm_source=chatgpt.com)
**Date:** 24/09/2026

---

## 🧭 Introduction

**ContainMe** is a particularly interesting TryHackMe machine because the objective isn't simply:

```
Web → root → flag
```

Instead, the machine makes you move through **multiple isolated environments**.

The overall chain is:

```
Web enumeration
      ↓
Hidden PHP parameter
      ↓
Command injection
      ↓
www-data on host1
      ↓
SUID binary
      ↓
root inside host1
      ↓
Discover internal network
      ↓
SSH into 172.16.20.6
      ↓
mike
      ↓
Internal MySQL
      ↓
Root credentials
      ↓
root on second container
      ↓
Extract protected archive
      ↓
FLAG
```

The room's own description boils the objective down to finding the hidden flag and emphasizes looking beyond the first environment.

---

# 🔍 1. Reconnaissance

I began with a full Nmap scan:

```
nmap -sC -sV -p- <TARGET_IP>
```

The machine exposes four TCP services. Independent scans of the room consistently identify:

```
22/tcp
80/tcp
2222/tcp
8022/tcp
```

with SSH services on multiple ports and HTTP on port 80.

At first glance, the SSH ports look interesting, but there aren't credentials yet, so I started with the web server.

---

# 🌐 2. Web Enumeration

Opening:

```
http://<TARGET_IP>/
```

only produced the default Apache page.

I therefore enumerated the web server:

```
gobuster dir -u http://<TARGET_IP>/ \
-w /usr/share/wordlists/dirb/common.txt \
-x php,html,txt
```

Among the interesting results were:

```
/index.php
/info.php
```

The `info.php` page exposed PHP information, while `/index.php` was much more interesting.

Instead of displaying a normal webpage, it returned a directory listing.

The listing showed files such as:

```
index.html
index.php
info.php
```

More importantly, the files were shown as being owned by `root`.

---

# 🕵️ 3. The Hidden Clue

I inspected the source of `/index.php`.

Buried in the HTML was:

```
<!-- where is the path ? -->
```

This is an enormous clue.

The page was already displaying the contents of a directory, so the obvious question became:

> Is there a parameter controlling which directory is being listed?

I tested the page with a `path` parameter:

```
http://<TARGET_IP>/index.php?path=/
```

The response changed according to the supplied path.

That confirmed that `path` was being processed by the application.

Independent walkthroughs confirm that the vulnerable parameter is indeed named `path`.

---

# 💥 4. Discovering Command Injection

At first, the functionality appears to be something similar to:

```
ls -la <supplied path>
```

For example:

```
/index.php?path=/home
```

could produce the contents of `/home`.

That means the application is effectively taking attacker-controlled input and inserting it into a shell command.

I tested whether shell metacharacters could terminate the original command.

The important character was:

```
;
```

A semicolon allows one command to end and another command to begin.

Conceptually:

```
ls -la <input>
```

could become:

```
ls -la / ; <attacker command>
```

This turned the directory-listing functionality into **arbitrary command execution**.

Other walkthroughs independently confirm that `;` is the key separator and that `&&`/`||` aren't the useful path here.

---

# 🐚 5. Getting the Initial Shell

Now that command execution was confirmed, the next objective was to turn it into an interactive shell.

A reverse-shell payload can be executed through the vulnerable parameter, with the attacker's listener waiting for the connection.

After the connection arrived, I checked:

```
whoami
```

and obtained:

```
www-data
```

So the initial foothold was:

```
www-data
```

The machine's hostname was particularly important:

```
hostname
```

returned:

```
host1
```

That name becomes extremely important later.

I then stabilized the shell so normal terminal interaction was possible.

---

# 🔎 6. Local Enumeration

With access to `host1`, I started looking for privilege-escalation opportunities.

One of the standard checks is for SUID files:

```
find / -type f \
\( -perm -u+s -o -perm -g+s \) \
-exec ls -la {} \; 2>/dev/null
```

Among the results was something unusual:

```
/usr/share/man/zh_TW/crypt
```

with the SUID bit set.

The location itself was suspicious.

A SUID executable runs with the privileges of its owner, regardless of the privileges of the user launching it.

Here, the important relationship was:

```
crypt
 ↓
owned by root
 ↓
SUID
 ↓
can execute with root privileges
```

There was also another copy of the program in Mike's home directory:

```
/home/mike/1cryptupx
```

The latter wasn't SUID, but it provided a useful copy for analysis.

---

# 🧩 7. Understanding the `crypt` Binary

Running the SUID program normally produced an error resembling:

```
Unable to decompress.
```

That was unusual.

Further investigation showed that the binary had been packed using **UPX**. Other independent ContainMe writeups also identify the binary as a UPX-packed executable.

The really interesting discovery came from experimenting with its argument.

The only useful username discovered so far was:

```
mike
```

Passing that to the SUID program caused a completely different result.

The program spawned a shell with root privileges.

So the first privilege escalation became:

```
www-data
     ↓
SUID crypt
     ↓
root
```

---

# 👑 8. But This Isn't the Final Root

This is where **ContainMe** becomes interesting.

Running:

```
whoami
```

now returns:

```
root
```

but:

```
hostname
```

still returns:

```
host1
```

This tells us that we've become root **inside the current environment**, not necessarily the final machine containing the flag.

I checked the network configuration:

```
ifconfig
```

or:

```
ip addr
```

An additional private interface was present:

```
172.16.20.2
```

That strongly suggested another network/container behind the current environment.

This is the point where the room changes from conventional privilege escalation into **network pivoting**.

Independent walkthroughs describe exactly this discovery: `host1` exposes a second interface on the `172.16.20.0/24` network.

---

# 🗺️ 9. Discovering the Second Host

The current environment didn't necessarily have all the tools needed for network enumeration.

A static Nmap binary can be transferred to the authorized lab machine and used to inspect the internal network.

The important network was:

```
172.16.20.0/24
```

Scanning the network revealed another live host:

```
172.16.20.6
```

with:

```
22/tcp open
```

That was the missing piece.

We had discovered another SSH-accessible machine inside the internal network.

---

# 🔑 10. Finding Mike's SSH Key

Earlier enumeration had revealed Mike's home directory:

```
/home/mike
```

Inside it was:

```
.ssh/
```

and, importantly:

```
id_rsa
```

The private key could be copied out of the first environment.

However, attempting to use it against the original machine didn't work.

That initially looks like the key is useless.

But now we know there is another host:

```
172.16.20.6
```

So the key is worth testing there.

---

# 🚪 11. Pivoting to the Second Container

Using Mike's private key:

```
ssh -i id_rsa mike@172.16.20.6
```

successfully established an SSH session.

We now had:

```
host1
   ↓
internal network
   ↓
172.16.20.6
   ↓
mike
```

This is a crucial distinction.

The first root shell was **not the final destination**.

We had pivoted into another environment and obtained a shell as:

```
mike
```

Independent walkthroughs confirm that `172.16.20.6` is the second host and that Mike's SSH key is used to access it.

---

# 🗄️ 12. Discovering MySQL

Once logged into the second environment, I enumerated its running services.

An important internal service was:

```
3306/tcp
```

Port 3306 is normally associated with:

```
MySQL
```

The service was accessible internally.

I attempted to connect:

```
mysql -u mike -p
```

The password discovered for Mike was:

```
password
```

The credentials worked.

So we now had database access:

```
mike
   ↓
MySQL
```

---

# 🔎 13. Enumerating the Database

Inside MySQL, I enumerated the available databases:

```
SHOW DATABASES;
```

Then inspected the relevant database and its tables:

```
USE accounts;
SHOW TABLES;
```

The interesting table was:

```
users
```

I queried it:

```
SELECT * FROM users;
```

The database contained credentials for multiple accounts, including:

```
mike
root
```

This was the final credential-discovery step.

The important chain was now:

```
MySQL
 ↓
accounts
 ↓
users
 ↓
root credentials
```

Independent walkthroughs confirm that the database contains the credentials for both `mike` and `root`.

---

# 👑 14. Becoming Root on the Second Host

Using the recovered root credentials, I switched to the root account:

```
su root
```

After entering the recovered password:

```
whoami
```

returned:

```
root
```

This time, we were finally root in the **second environment**.

The distinction is:

```
FIRST ENVIRONMENT
www-data
   ↓
SUID exploit
   ↓
root on host1
        │
        │ pivot
        ▼
SECOND ENVIRONMENT
mike
   ↓
MySQL
   ↓
root
```

---

# 📦 15. Finding the Flag

I checked `/root`:

```
cd /root
ls -la
```

Among the files was an archive:

```
mike.zip
```

The archive was password protected.

The password recovered from the database could be used to unlock it.

After extracting the archive, the contents revealed the final flag.

---

# 🚩 16. Final Flag

The answer to the room's question:

> **What is the flag?**

is:

```
THM{_Y0U_F0UND_TH3_C0NTA1N3RS_}
```

🏁 **Flag**

```
THM{_Y0U_F0UND_TH3_C0NTA1N3RS_}
```

The same flag is independently reported by multiple ContainMe walkthroughs and matches the current TryHackMe room's single flag objective.

---

# 🗺️ Complete Attack Chain

Here's the whole room condensed into one diagram:

```
                    TARGET
                       │
                       ▼
                 Nmap enumeration
                       │
                       ▼
                 Apache :80
                       │
                       ▼
                  /index.php
                       │
                       ▼
             Source-code clue:
             "where is the path?"
                       │
                       ▼
                 path= parameter
                       │
                       ▼
               Command injection
                       │
                       ▼
                    www-data
                       │
                       ▼
                host1 identified
                       │
                       ▼
                SUID enumeration
                       │
                       ▼
              /usr/.../crypt
                       │
                       ▼
             SUID root execution
                       │
                       ▼
                 "mike" argument
                       │
                       ▼
                ROOT — host1
                       │
                       ▼
            Discover 172.16.20.2
                       │
                       ▼
          Scan 172.16.20.0/24
                       │
                       ▼
              172.16.20.6 :22
                       │
                       ▼
             Mike's id_rsa key
                       │
                       ▼
                SSH as mike
                       │
                       ▼
              SECOND CONTAINER
                       │
                       ▼
                 MySQL :3306
                       │
                       ▼
                mike : password
                       │
                       ▼
              accounts.users
                       │
                       ▼
               root credentials
                       │
                       ▼
                  ROOT — host2
                       │
                       ▼
                    /root
                       │
                       ▼
                   mike.zip
                       │
                       ▼
                Extract archive
                       │
                       ▼
                    FLAG
```

---

# 🧠 Lessons Learned

### 1. Source code comments can be attack clues

The comment:

```
<!-- where is the path ? -->
```

looked harmless, but it directly pointed toward the vulnerable parameter.

**Lesson:** Always inspect HTML source, especially when a challenge gives you an unusual or incomplete page.

### 2. Don't assume a directory listing is harmless

The application appeared to simply run something equivalent to:

```
ls -la
```

But because attacker-controlled input was inserted into the command, the functionality became command injection.

The difference between:

```
listing a directory
```

and:

```
executing attacker-controlled commands
```

was just insufficient input validation.

### 3. SUID binaries deserve serious attention

The unusual:

```
/usr/share/man/zh_TW/crypt
```

binary was the first major escalation opportunity.

Its combination of:

```
root ownership
+
SUID
+
attacker-controlled argument
```

made it extremely valuable.

### 4. Root doesn't necessarily mean game over

This is probably the most important lesson from **ContainMe**.

Getting:

```
uid=0(root)
```

doesn't automatically mean you've reached the machine containing the flag.

The hostname:

```
host1
```

and the additional interface:

```
172.16.20.2
```

were clues that another environment existed.

### 5. Containers create additional attack surfaces

The machine intentionally makes you move between isolated environments.

The first environment gives you access to:

```
172.16.20.0/24
```

which exposes:

```
172.16.20.6
```

The second environment then contains another service and another privilege-escalation path.

That's why the room is called **ContainMe**.

### 6. Internal services can be more important than exposed ones

The MySQL service wasn't the initial external attack surface.

It became useful only **after pivoting into the internal environment**.

This is an important network-security concept:

```
Internet-facing host
        ↓
Compromise
        ↓
Internal network access
        ↓
Previously unreachable service
        ↓
Further compromise
```

### 7. Credential reuse creates chains

The room repeatedly demonstrates why credentials shouldn't be reused.

We encounter:

```
mike
 ↓
SSH key
 ↓
SSH access
 ↓
MySQL password
 ↓
root credentials
```

Each discovery unlocks another part of the environment.

---

# 🎯 Conclusion

**ContainMe** isn't really one privilege-escalation challenge. It's a **pivoting challenge disguised as a simple web application**.

The first half teaches:

> **Web enumeration → command injection → SUID exploitation**

The second half teaches:

> **Container awareness → network enumeration → SSH pivot → internal service enumeration → credential discovery**

And the complete chain is:

> **Apache → `path` command injection → `www-data` → SUID `crypt` → root on `host1` → `172.16.20.6` → `mike` → MySQL → root on the second container → `mike.zip` → flag**

### 🏆 Final Flag

```
THM{_Y0U_F0UND_TH3_C0NTA1N3RS_}
```