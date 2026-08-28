# Overpass

**Platform:** TryHackMe
**Date:** date:2026-08-28
**Difficulty:** Easy
**Category:** Privilege Escalation/Web Hacking
**Room URL:** https://tryhackme.com/room/overpass
**Status:** ✅ Completed

---

## 📌 What Was This Room About?
> This room was about using your web hacking and bypass auth skills to hack into a password manager made by broke computer science students. Then after hacking it using Privilege Escalation to gain access to the root account to get the flag.

---

## 🔍 Reconnaissance
> What was the first thing you did? Always start with recon.

### Nmap Scan
```bash
# Replace with your actual command
nmap -sV -sC -oN 10.129.142.232
```

**Results:**
![[Screenshot 2026-03-27 at 6.24.03 PM.png]]
---

## 🧭 Steps I Took

> Write each step in order. Explain WHY you did it, not just WHAT you did.
### Step 1 — 
- **What I did:** The first thing I did was obviously dig around in the source code a bit to try to find any clues. I was not successful at finding anything interesting so I went on with my normal method by booting up [[dirb]] to find any hidden directories. The most interesting directory was the /admin directory and when I went there I found a login page. I tried a bunch of more things but I when I exhausted everything I got, I sought help.
- **Command/Tool used:**
```bash
# dirb 10.129.142.232
```
- **What I found:** I found that this challenge is going to be much harder than the other challenges. 

### Step 2 — 
- **What I did:** I tried a few more things but I when I exhausted everything I got, I sought help. I used this tutorial by John Hammond: https://www.youtube.com/watch?v=NGNnxD0gNDw which was very helpful to a point. I followed him and found out that when you type in this command in the Inspector console: `Cookies.set("SessionToken", "anything")` I got into this page: ![[Screenshot 2026-03-27 at 6.37.13 PM.png]]

- **Command/Tool used:**
```bash
# command here
```
- **What I found:**
- **Why it matters:**

### Step 3 — 
- **What I did:**
- **Command/Tool used:**
```bash
# command here
```
- **What I found:**
- **Why it matters:**

> ➕ Add more steps as needed by copying the block above.

---

## 🛠 Commands Used

| Command | What It Does |
|---------|--------------|
|         |              |
|         |              |
|         |              |

---

## 🚩 Flags Found

| Flag | Value |
|------|-------|
| User flag | `THM{...}` |
| Root flag | `THM{...}` |

---

## 💡 What I Learned
> Most important section. Write at least 3 bullet points.

- 
- 
- 

---

## ❓ What Confused Me / What to Research Next
> Be honest. This is where real learning happens.

- 
- 

---

## 🔗 Linked Notes
> Connect this report to your concept and tool notes.

- [[Nmap]]
- [[Linux Commands]]
- [[Add more links as relevant]]

---

## 📎 Resources Used
> Links to tutorials, writeups, or documentation that helped you.

- 

---
*Report written in [[TryHackMe Vault]] — part of my ethical hacking journey 🛡️*
