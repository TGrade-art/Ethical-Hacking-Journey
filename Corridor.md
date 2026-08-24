# Corridor

**Platform:** TryHackMe  
**Date:** 2026-08-24  
**Difficulty:** Easy  
**Category:** Web / IDOR / Cryptography  
**Room URL:** [TryHackMe Corridor](https://tryhackme.com/room/corridor?utm_source=chatgpt.com)  
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

This room was about finding an **IDOR (Insecure Direct Object Reference)** vulnerability hidden behind MD5 hashes. The website looked like it was using random-looking hashes for the different rooms, but the hashes were actually just predictable numbers hashed with MD5.

---

## 🔍 Reconnaissance

First things first: recon.

I started by opening the target and checking what was running on it.

### Nmap Scan

```bash
nmap -sV -sC -oN nmap_output.txt <TARGET_IP>
```

**Results:**

|Port|Service|Version|Notes|
|---|---|---|---|
|80|HTTP|Werkzeug/Python|Hosts the Corridor web application|

The website itself was pretty simple. I was presented with a corridor containing **13 different doors**. Clicking on the doors took me to different URLs.

That was when I started paying attention to the URL instead of just the webpage.

---

## 🧭 Steps I Took

### Step 1 — Checking the Door URLs

- **What I did:**  
    I clicked on different doors and watched what happened to the URL.
    
- **Command/Tool used:**  
    Web browser.
    
- **What I found:**  
    Every door added a long 32-character hexadecimal value to the URL.
    
- **Why it matters:**  
    A 32-character hexadecimal string immediately made me think of **MD5**. So instead of assuming these were random values, I wanted to figure out what they actually represented.
    

---

### Step 2 — Figuring Out the Hashes

- **What I did:**  
    I inspected the HTML and found the hashes being used for the different doors. I then tested them against hash databases/tools.
    
- **Command/Tool used:**  
    CrackStation / CyberChef / Python.
    
- **What I found:**  
    The hashes weren't anything complicated. They were simply MD5 hashes of numbers from **1 to 13**.
    
    The doors followed this pattern:
    

```text
1 2 3 4 5 6 7 13 12 11 10 9 8
```

- **Why it matters:**  
    This was the big clue. The application wasn't actually using secure random identifiers. It was just hashing predictable numbers.
    
    So if I could calculate the MD5 hash of another number, I could potentially access another page without clicking through the interface.
    

---

### Step 3 — Testing Outside the Normal Range

- **What I did:**  
    Since the doors represented numbers from 1–13, I started testing numbers outside that range.
    
- **Command/Tool used:**  
    MD5 generator + browser.
    

For example:

```bash
echo -n "14" | md5sum
```

gave the MD5 hash for `14`.

I also tested `0`:

```bash
echo -n "0" | md5sum
```

which gave:

```text
cfcd208495d565ef66e7dff9f98764da
```

- **What I found:**  
    `14` didn't work, but `0` did.
    
- **Why it matters:**  
    This was the actual IDOR. The website trusted the object identifier supplied in the URL instead of properly checking whether that object was supposed to be accessible.
    
    Going directly to the MD5 hash of `0` caused the server to return a hidden page containing the flag.
    

And that's basically where the room was solved. 😂

---

## 🛠 Commands Used

|Command / Tool|What It Does|
|---|---|
|`nmap -sV -sC <TARGET_IP>`|Scans the target and identifies services and versions|
|`echo -n "0" \| md5sum`|Generates the MD5 hash of `0`|
|CrackStation|Helps identify known hashes|
|Browser Inspector (F12)|Used to inspect the HTML and find the door/hash values|

---

## 🚩 Flags Found

| Flag      | Value                                    |
| --------- | ---------------------------------------- |
| Room Flag | `flag{2477ef02448ad9156661ac40a6b8862e}` |

---

## 📎 Resources Used

- [OWASP Top 10 — Broken Access Control](https://owasp.org/Top10/A01_2021-Broken_Access_Control/?utm_source=chatgpt.com)
    
- [CrackStation Hash Database](https://crackstation.net/?utm_source=chatgpt.com)
    

---

