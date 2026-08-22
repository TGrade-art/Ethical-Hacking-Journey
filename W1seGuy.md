# XOR Cryptography Challenge (W1seGuy)

**Platform:** TryHackMe  
**Date:** 2026-08-22  
**Difficulty:**  Easy
**Category:** Cryptography / Networking / Python  
**Room URL:** [https://tryhackme.com/room/lsegguy](https://tryhackme.com/room/lsegguy)  
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> This challenge was about breaking XOR encryption. The target had a network service running on port `1337` that gave me hex-encoded ciphertext and key hints, so I had to figure out how the encryption worked, write a little Python script to decrypt it, and eventually pull out both flags. 🔐

---

## 🔍 Reconnaissance

### Finding the Target Service

The lab machine was running a service on TCP port `1337`.

**Target Machine:** `10.129.143.81`  
**Attacker Machine:** `10.129.71.166`

I connected to the service with Netcat to see what it would give me:

```bash
nc 10.129.143.81 1337
```

The service returned XOR-encoded text and asked me for the encryption key.

Example:

```text
This XOR encoded text has flag 1: 032794a1266ed4851f78f029375e01603f86e9293b2f5655735f048d117253f431f
What is the encryption key? Close but no cigar
```

That wasn't enough to immediately solve it, so I connected again.

---

## 🧭 Steps I Took

### Step 1 — Connecting to the XOR Service

- **What I did:** Connected to the service running on port `1337` using Netcat.
    
- **Command/Tool used:**
    

```bash
nc 10.129.143.81 1337
```

- **What I found:** The service returned a hex-encoded string and asked for an encryption key.
    
- **Why it matters:** This told me the actual challenge was happening through a network service rather than a normal web application. I needed to understand what was generating the ciphertext before trying to crack it.
    

---

### Step 2 — Getting a Useful Key Hint

I connected to the service again and got a different ciphertext along with a much more useful response:

```text
This XOR encoded text has flag 1: 351d080315893d351d24232d2132351561361822093b2740290d192c1b141322c433432d1a3c
What is the encryption key? A0U5A
```

The important part here was:

```text
Ciphertext:
351d080315893d351d24232d2132351561361822093b2740290d192c1b141322c433432d1a3c

Key:
A0U5A
```

Now I had something I could actually work with.

---

### Step 3 — Looking at the Source Code

The challenge also provided a Python source file.

Looking through it showed that the flag was XORed against a repeating key:

```python
flag = open('flag.txt', 'r').read().strip()

xored = []

for i in range(0, len(flag)):
    xored += chr(ord(flag[i]) ^ ord(key[i % len(key)]))

hex_encoded = "".join(xored).encode().hex()
```

This was the important part.

The encryption was basically:

```text
plaintext XOR repeating key = ciphertext
```

Since XOR is reversible:

```text
ciphertext XOR key = plaintext
```

So I didn't need to somehow "undo" a complicated encryption algorithm. I just needed to XOR the ciphertext with the correct key.

---

### Step 4 — Writing the Decryption Script

Rather than manually XORing everything, I made a Python script called `decode.py` to handle it for me.

The basic logic was:

```python
decrypted = ""

for i in range(len(ciphertext)):
    decrypted += chr(
        ord(ciphertext[i]) ^
        ord(full_key[i % len(full_key)])
    )
```

I also added a check for the expected flag format so the script could tell me when it found something that looked like a real flag.

```text
THM{...}
```

I then ran:

```bash
python3 decode.py
```

And got:

```text
[+] Found Key: A0U5A
[+] Decrypted Flag: THM{plaintextt4ckc4nB3A3llyHurdty0urX0r}
```

BOOM. 🔓

---

### Step 5 — Decrypting the Second Flag

The service generates new encrypted data, so I repeated the Netcat/decryption process for the second challenge.

Eventually I recovered the second flag:

```text
THM{Brut3F0rC1ng_X0R_c4n_B3_Fuk_n0T}
```

The main idea stayed the same: obtain the ciphertext and key information, then use XOR to recover the plaintext.

---

## 🛠 Commands Used

|Command / Tool|What It Does|
|---|---|
|`nc 10.129.143.81 1337`|Connects to the XOR challenge service|
|`python3 decode.py`|Runs my XOR decryption script|
|`ord()`|Converts characters into integer values|
|`chr()`|Converts decrypted integer values back into characters|
|`^`|Python's XOR operator|
|`hex()` / `.encode().hex()`|Handles hexadecimal representation of the ciphertext|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|Flag 1|`THM{plaintextt4ckc4nB3A3llyHurdty0urX0r}`|
|Flag 2|`THM{Brut3F0rC1ng_X0R_c4n_B3_Fuk_n0T}`|

---

## 💡 What I Learned

- **XOR is extremely reversible.** If you know the key, decrypting XOR is basically just doing XOR again.
    
- **Understanding the source code was more useful than blindly attacking the service.** Once I saw `flag[i] ^ key[i % len(key)]`, I knew exactly what I needed to reproduce.
    
- **Automating repetitive work with Python is way better than doing it manually.** Instead of trying different values by hand, I made `decode.py` handle the decryption and flag checking.
    
- **Hex encoding isn't encryption.** The ciphertext looked complicated at first, but the hex was just another representation of the XOR output.
    
- **Sometimes the easiest way to solve a crypto challenge is to understand exactly how the encryption was implemented.** Once I knew the algorithm, the challenge became much simpler.
    

---

## ❓ What Confused Me / What to Research Next

- I want to get more comfortable with **XOR cryptanalysis**, especially situations where the key isn't directly given.
    
- I also want to learn more about **known-plaintext attacks** and how repeating-key XOR can be attacked when you don't know the key.
    
- I'd like to build more of my own crypto scripts instead of relying on existing tools.
    

---

## 🔗 Linked Notes

- [XOR Cryptography](https://chatgpt.com/c/XOR%20Cryptography)
    
- [Python](https://chatgpt.com/c/Python)
    
- [Cryptography](https://chatgpt.com/c/Cryptography)
    
- [Netcat](https://chatgpt.com/c/Netcat)
    
- [Hexadecimal](https://chatgpt.com/c/Hexadecimal)
    
- [TryHackMe](https://chatgpt.com/c/TryHackMe)
    

---

## 📎 Resources Used

- TryHackMe — `lsegguy`
    
- Python documentation
    
- XOR / bitwise operation documentation
    

---

_Report written in_ [_TryHackMe Vault_](https://chatgpt.com/c/TryHackMe%20Vault) _— part of my ethical hacking journey 🛡️_