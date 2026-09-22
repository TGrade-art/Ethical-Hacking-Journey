# Silent Monitor

**Room:** [TryHackMe — Silent Monitor](https://tryhackme.com/room/silent-monitor)
**Difficulty:** Medium
## Overview

**Silent Monitor** is a Linux penetration-testing challenge that demonstrates how several relatively simple vulnerabilities can be chained together to completely compromise a machine.

The attack begins with network enumeration and web discovery, then moves through:

```text
Network Enumeration
        ↓
Web Application Discovery
        ↓
Hidden /internal Portal
        ↓
SQL Injection Authentication Bypass
        ↓
Command Injection
        ↓
Credential Discovery
        ↓
SSH Access
        ↓
KeePass Database Discovery
        ↓
Password Cracking
        ↓
Root Credential Recovery
        ↓
Root Access
```

The room is particularly useful because the vulnerabilities aren't isolated. Information discovered during one stage becomes the key to the next stage.

TryHackMe currently lists Silent Monitor as part of its **Jr Penetration Tester Challenges**, alongside other end-to-end challenges designed to combine reconnaissance, exploitation, and privilege escalation skills.

---

# Task 1 — Reconnaissance

## Full Port Scan

I started by scanning the target for open TCP ports and service versions.

```bash
nmap -sC -sV -p- <MACHINE_IP> -Pn
```

### Why `-Pn`?

The target may not respond normally to ICMP host discovery, so Nmap can incorrectly assume that the machine is offline.

`-Pn` tells Nmap to skip the normal host-discovery stage and treat the target as online.

The scan revealed two important services:

```text
22/tcp    SSH
5050/tcp  HTTP
```

The HTTP service was running a Python-based application using Werkzeug.

The important observation here is that **port 5050**, rather than the normal HTTP port 80, hosts the web application.

So the next step was:

```text
http://<MACHINE_IP>:5050
```

The main page itself did not expose much functionality, so I moved on to web-content enumeration.

---

# Task 2 — Web Enumeration

I used directory enumeration against the web server:

```bash
dirsearch -u http://<MACHINE_IP>:5050
```

An interesting endpoint appeared:

```text
/internal
```

Visiting:

```text
http://<MACHINE_IP>:5050/internal
```

revealed an internal login portal.

This immediately became the most interesting part of the application.

A hidden endpoint isn't automatically secure simply because it isn't linked from the homepage. If an attacker can discover it, the authentication mechanism itself still needs to properly validate and handle untrusted input.

---

# Task 3 — Authentication Bypass

The `/internal` page contained username and password fields.

I first considered normal authentication testing, but the application appeared to be interacting with a backend database.

That made **SQL injection** a natural test.

A basic authentication-bypass payload is:

```text
' OR 1=1 -- -
```

The idea behind the payload is to alter the SQL query's logical condition so that the authentication check evaluates as true.

The important lesson isn't simply memorizing the payload. The real vulnerability is that the application is apparently incorporating user-controlled data into a SQL query instead of safely parameterizing it.

The injection successfully bypassed the login and provided access to the internal dashboard.

---

# Task 4 — Exploring the Internal Dashboard

After authentication, the dashboard exposed functionality for checking the health/connectivity of another host.

This immediately stood out.

A feature like:

```text
Host Health
```

often works by executing a system command such as:

```bash
ping <user_input>
```

If the application constructs that command using unsanitized user input, the feature can become a **command-injection vulnerability**.

Rather than assuming the feature was vulnerable, I inspected how the request was being sent using **Burp Suite**.

---

# Task 5 — Command Injection

The vulnerable parameter was the host/target value.

One important detail discovered during testing was that the web interface encoded certain characters more than once.

For example, a newline character represented as:

```text
%0A
```

could become:

```text
%250A
```

after another layer of URL encoding.

That prevents the intended payload from reaching the vulnerable backend in the expected form.

Using Burp Suite to modify the raw request allowed the encoded newline to reach the application correctly.

A test request followed the general pattern:

```text
target=127.0.0.1%0A<command>
```

For example, listing the `/home` directory demonstrated command execution:

```text
target=127.0.0.1%0A ls /home
```

The response showed accounts including:

```text
sysadmin
ubuntu
```

This confirmed that the input was not simply being used as an IP address — additional operating-system commands were being executed.

The vulnerability can therefore be classified as **OS command injection**.

---

# Why the Command Injection Worked

Conceptually, the vulnerable application was performing something equivalent to:

```text
ping <user-controlled-value>
```

The application expected the user to provide something like:

```text
127.0.0.1
```

But insufficient input separation allowed the attacker-controlled value to influence the shell command itself.

This is why command injection is particularly dangerous: the application may appear to provide a harmless feature such as "check whether this host is online," while actually giving the attacker a path to execute commands as the web application's operating-system user.

### Defensive Fixes

Developers should:

- Avoid shell commands when a native API can perform the same function.
    
- Validate IP addresses and hostnames against strict allowlists.
    
- Never concatenate raw user input into shell commands.
    
- Avoid unnecessary shell interpretation.
    
- Run the application with minimal privileges.
    
- Log suspicious command-injection attempts.
    

---

# Task 6 — Finding `secret.config`

Once command execution was confirmed, I began enumerating files and directories accessible to the application.

One particularly interesting directory was:

```text
/opt/netops
```

Listing it revealed several application files, including:

```text
app.py
netops.db
secret.config
templates/
```

The file that immediately stood out was:

```text
secret.config
```

Configuration files are worth investigating because they can contain sensitive information such as:

- Database credentials
    
- Service-account credentials
    
- API keys
    
- Backup passwords
    
- Encryption keys
    
- Internal usernames
    

Reading the file through the command-injection vulnerability exposed credentials for the `sysadmin` account.

This was the turning point from **web application compromise** to **system access**.

---

# Task 7 — SSH Access

During the initial Nmap scan, port 22 was discovered running SSH.

Now we had credentials for a system account, so the previously discovered SSH service became immediately useful.

The general connection syntax is:

```bash
ssh sysadmin@<MACHINE_IP>
```

The credentials obtained from `secret.config` successfully authenticated.

We now had an interactive shell as:

```text
sysadmin
```

This demonstrates why good reconnaissance matters.

At the beginning of the attack, SSH was simply an exposed service. After discovering credentials elsewhere on the machine, that same service became the gateway to the operating system.

---

# Task 8 — User Flag

After obtaining the SSH shell, I enumerated the user's home directory:

```bash
ls -la
```

The user flag was located at:

```text
/home/sysadmin/user.txt
```

It can be retrieved with:

```bash
cat ~/user.txt
```

The room's user flag is:

```text
THM{sQli_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}
```

This flag is a nice summary of the first half of the attack: the initial compromise relied on **SQL injection followed by command injection**.

---

# Task 9 — Investigating the Backups

The next objective was obtaining root access.

While enumerating the `sysadmin` home directory, a directory named:

```text
backups
```

stood out.

Inside it was a KeePass database:

```text
infrastructure.kdbx
```

There was also a README explaining the purpose of the database.

The `.kdbx` extension is associated with **KeePass password databases**.

This was significant because a password database could contain credentials for privileged accounts.

Instead of attempting to guess the root password directly, the next step was to recover the password protecting the KeePass database.

---

# Task 10 — Extracting the KeePass Hash

First, transfer the database from the TryHackMe machine to the attacker machine.

For example, from the attacker machine:

```bash
scp sysadmin@<MACHINE_IP>:/home/sysadmin/backups/infrastructure.kdbx .
```

Once the file has been downloaded, identify its type:

```bash
file infrastructure.kdbx
```

It should identify the file as a KeePass database.

A common approach for older supported KeePass formats is to use:

```bash
keepass2john infrastructure.kdbx > hash.txt
```

Then the resulting hash can be tested with John the Ripper:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

Depending on the KDBX version, however, `keepass2john` may not support the database format. Current Silent Monitor write-ups report that the database can require a KeePass-specific cracking tool capable of handling the newer format.

The important methodological point is:

```text
KDBX file
   ↓
Extract/crack the database password
   ↓
Open the KeePass database
   ↓
Recover privileged credentials
```

---

# Task 11 — Opening the KeePass Database

After recovering the KeePass master password, open the database using a compatible KeePass client.

The database contains stored credentials, including credentials for the root account.

This is a good example of why encrypted password databases should themselves be protected carefully.

The `.kdbx` file was encrypted, but the security of the entire database ultimately depended on the strength of its master password.

---

# Task 12 — Root Access

With the root credentials recovered from the KeePass database, switch to the root account:

```bash
su root
```

Enter the recovered root password.

If successful, verify the current identity:

```bash
id
```

The result should show:

```text
uid=0(root)
```

The final flag can then be read from:

```bash
cat /root/root.txt
```

The room's root flag is:

```text
THM{KDBx_V4ul7_H4s_b33n_cr4ck3d_0peN}
```

---

# Complete Attack Chain

The complete compromise can be visualized as:

```text
                Nmap
                  │
                  ▼
       ┌─────────────────────┐
       │ SSH + HTTP :5050    │
       └──────────┬──────────┘
                  │
                  ▼
          Directory Discovery
                  │
                  ▼
             /internal
                  │
                  ▼
          SQL Injection
                  │
                  ▼
        Internal Dashboard
                  │
                  ▼
          Host Health Tool
                  │
                  ▼
        Command Injection
                  │
                  ▼
          secret.config
                  │
                  ▼
       sysadmin Credentials
                  │
                  ▼
              SSH
                  │
                  ▼
          sysadmin Shell
                  │
                  ▼
        /home/sysadmin/backups
                  │
                  ▼
       infrastructure.kdbx
                  │
                  ▼
       Crack KDBX Password
                  │
                  ▼
       Recover Root Credentials
                  │
                  ▼
              su root
                  │
                  ▼
             root.txt
```

---

# Vulnerabilities Identified

|Vulnerability|Location|Impact|
|---|---|---|
|SQL Injection|`/internal` login|Authentication bypass|
|Command Injection|Host Health functionality|Arbitrary command execution|
|Plaintext credentials|`secret.config`|SSH account compromise|
|Sensitive backup exposure|`infrastructure.kdbx`|Credential disclosure|
|Weak KeePass protection|KDBX database|Recovery of privileged credentials|
|Credential reuse/exposure|SSH/root credentials|Privilege escalation|

---

# Defensive Lessons

## 1. Use Parameterized SQL Queries

The login vulnerability exists because user-controlled input can influence the SQL query.

Applications should use parameterized queries or prepared statements rather than constructing SQL statements from raw input.

---

## 2. Never Pass Raw Input to a Shell

The host-health feature should never construct commands by concatenating user input.

If a ping operation is required, the application should use a safe process-execution API and strictly validate the supplied address.

---

## 3. Protect Configuration Files

`secret.config` contained credentials that ultimately allowed SSH access.

Sensitive configuration files should:

- Have restrictive permissions.
    
- Avoid plaintext credentials where possible.
    
- Use environment-specific secret-management systems.
    
- Never expose unnecessary credentials to the web application's account.
    

---

## 4. Protect Backup Files

The KeePass database was stored inside a user's backup directory.

Backups can contain extremely sensitive information and therefore need the same security considerations as production data.

Access should be restricted according to least privilege.

---

## 5. Use Strong Passwords for Password Vaults

A password manager provides strong protection only when the master password is sufficiently resistant to offline guessing.

A stolen encrypted vault can be attacked offline, meaning there is no web login rate limit to slow an attacker down.

---

## 6. Avoid Credential Reuse

The attack became much easier because credentials discovered during one stage could be used to access another service.

Different services and privilege levels should use separate credentials.

---

# Key Commands

### Network enumeration

```bash
nmap -sC -sV -p- <MACHINE_IP> -Pn
```

### Directory enumeration

```bash
dirsearch -u http://<MACHINE_IP>:5050
```

### SSH

```bash
ssh sysadmin@<MACHINE_IP>
```

### Local enumeration

```bash
id
ls -la
```

### KeePass hash extraction

```bash
keepass2john infrastructure.kdbx > hash.txt
```

### John the Ripper

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

### Root access

```bash
su root
```

### Root flag

```bash
cat /root/root.txt
```

---

# Final Answers

|Objective|Answer|
|---|---|
|User flag|`THM{sQli_4nd_cMd_1nj3ct10n_l3D_y0u_h3re!}`|
|Root flag|`THM{KDBx_V4ul7_H4s_b33n_cr4ck3d_0peN}`|

## Conclusion

Silent Monitor is a strong example of **attack-path chaining**.

The first vulnerability alone only bypassed authentication. The second gave command execution, but that execution initially occurred within the web application's context. The discovered configuration file then supplied credentials that turned web access into SSH access. Finally, the exposed KeePass database provided the credentials necessary to obtain root.

The important lesson is therefore not simply _"find SQL injection"_ or _"find a KeePass file."_

It is the ability to continually ask:

> **"What can I do with the information I just discovered?"**

That mindset is what turns individual findings into a complete penetration-testing attack chain.