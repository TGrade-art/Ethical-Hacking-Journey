# TryHackMe Notes
**Date:** 2026-04-09
**Tags:** #tryhackme #linux #ctf #ethical-hacking #ftp #ssh #privilege-escalation

---

## 🚩 CTF Challenge: Team (TryHackMe)

### Room Summary
- **Room Name:** Team
- **Objective:** Gain access to the target machine using any available means and capture the flags
- **Flags Found:** ✅ `user.txt` | ✅ `root.txt`

---

### 🔍 Phase 1 — Reconnaissance

**Nmap Scan**
Started with an Nmap scan to identify open ports and running services:
```bash
nmap -sV TARGET_IP
```

**Open ports discovered:**
| Port | Service | Notes |
|---|---|---|
| 21 | FTP | Running an outdated version |
| 22 | SSH | Running an outdated version |
| 80 | HTTP (Apache 2) | Running an outdated version |

All three services were running **outdated versions**, indicating potential vulnerabilities.

---

### 🌐 Phase 2 — Web Enumeration

**Directory Busting with Dirb**
Used `dirb` to find hidden directories on the web server:
```bash
dirb http://TARGET_IP
```
- Found `/assets` directory — initially overlooked
- Found a file called `Rick` in the assets section — turned out to be a Rickroll planted by the room creator *(got Rickrolled again — this was the second time, the first being by TryHackMe itself)*

**Directory Busting with GoBuster**
Switched to GoBuster for more advanced enumeration:
```bash
gobuster dir -u http://TARGET_IP -w /path/to/wordlist
```
- Found `/css` directory
- Inside `/css`, discovered a hidden **image file**

---

### 🖼️ Phase 3 — Steganography & String Extraction

The image's presence in the `/css` folder was suspicious, so steganography was considered.

**Attempted: StegSeek**
```bash
stegseek image.jpg
```
- Failed due to formatting issues and an outdated virtual machine environment

**Used Instead: `strings`**
```bash
strings image.jpg
```
- Output contained mostly garbage values
- Found a **hidden comment** embedded in the file containing:
  - **Username:** `ftpuser`
  - A list of possible passwords (mixed in with garbage/junk strings)

---

### 🔐 Phase 4 — FTP Brute Force with Hydra

Since the password couldn't be identified manually from the noise, used **Hydra** to brute-force the FTP login.

**Steps:**
1. Copied all possible passwords from the `strings` output into a custom wordlist file called `war`
2. Ran Hydra against the FTP port:

```bash
hydra -l ftpuser -P war ftp://TARGET_IP
```
- Successfully cracked the FTP password

**FTP Login:**
```bash
ftp TARGET_IP
# Username: ftpuser
# Password: [cracked]
```

---

### 📂 Phase 5 — FTP Exploration & Credential Discovery

Once inside the FTP server, found a file. Downloaded it locally since `cat` is not available in FTP:
```bash
get filename
```

- The file was **encrypted using basic cryptography**
- Decrypted it manually using cryptographic knowledge
- Contents revealed:
  - **Username:** `ellie`
  - **Password:** `[decrypted value]`

---

### 🖥️ Phase 6 — SSH Access (Initial Foothold)

Used the discovered credentials to SSH into the machine:
```bash
ssh ellie@TARGET_IP
```
- Prompted for password — entered successfully
- **Gained access** to the machine ✅
- However, this was a **non-privileged account** — could not read protected files yet

---

### 🏳️ Phase 7 — Flag 1 (user.txt) via Privilege Escalation

On the machine, found a prompt reading: **"For Gwendolyn's eyes only"**
- Referenced a file with a name containing numbers and letters (obfuscated filename)

**Located the file using `find`:**
```bash
find / -name "*secret*" 2>/dev/null
```
- Found in: `/usr/games/`

**Read the file:**
```bash
cat /usr/games/[filename]
```
- Contained **Gwendolyn's username and password**

**Switched to Gwendolyn's account using `su`:**
```bash
su gwendolyn
# Password: [found in file]
```
- Now had the privileges needed to read `user.txt`
- **🏳️ Flag 1 captured: `user.txt`** ✅

---

### ⚡ Phase 8 — Root Shell via Vim GTFOBins (Privilege Escalation to Root)

Investigated privilege escalation vectors and found a **NOPASSWD sudo entry** allowing `vim` to be run with root privileges.

**Checked using:**
```bash
sudo -l
```

**Used [GTFOBins](https://gtfobins.github.io/)** to find the Vim exploit — ran the following inside a `sudo vim` session to spawn a root shell:
```vim
:!/bin/bash
```
or via shell escape within vim:
```bash
sudo vim -c ':!/bin/bash'
```

- Gained a **root shell** ✅
- Located and read `root.txt`
- **🏳️ Flag 2 captured: `root.txt`** ✅

---

## 🧠 Key Takeaways

- **Enumeration is everything** — both Dirb and GoBuster were needed to find all hidden directories
- **Steganography tools can fail** — `strings` is a reliable fallback when `stegseek` doesn't work
- **FTP can hold credentials** — always explore FTP thoroughly; files there can unlock SSH access
- **GTFOBins is essential** — always check for sudo permissions with `sudo -l` and cross-reference GTFOBins for privilege escalation paths
- **Vim with sudo = root shell** — a common and powerful escalation vector

## 🛠️ Tools Used

| Tool | Purpose |
|---|---|
| `nmap` | Port scanning & service detection |
| `dirb` | Web directory enumeration |
| `gobuster` | Advanced web directory enumeration |
| `strings` | Extract readable text from binary/image files |
| `stegseek` | Steganography tool (attempted) |
| `hydra` | Brute-force FTP login |
| `ftp` | Access FTP server |
| `ssh` | Remote login to target machine |
| `su` / `sudo` | Switch user / escalate privileges |
| `find` | Locate files on the filesystem |
| `cat` | Read file contents |
| `vim` | Text editor (also used for root shell escape) |
| GTFOBins | Reference for privilege escalation via binaries |
