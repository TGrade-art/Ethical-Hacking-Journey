# Crypto Failures

### Complete Walkthrough

**Room:** [TryHackMe — Crypto Failures](https://tryhackme.com/room/cryptofailures?utm_source=chatgpt.com)  
**Category:** Web / Cryptography  
**Difficulty:** Easy  
**Main vulnerability:** Broken custom authentication scheme using PHP `crypt()`
**Date:** 24/09/2026

The application attempts to create a "military-grade" authentication cookie, but its construction makes the cookie predictable and partially controllable. TryHackMe describes the room as requiring two stages: first breaking the authentication scheme, then recovering the encryption key.

---

# 1. Reconnaissance

First, add the target hostname to `/etc/hosts`:

```
sudo nano /etc/hosts
```

Add:

```
<TARGET_IP> cryptofailure.thm
```

Then perform a full TCP scan:

```
nmap -T4 -n -sC -sV -Pn -p- cryptofailure.thm
```

The scan reveals two relevant services:

```
22/tcp   open  ssh
80/tcp   open  http
```

So the attack surface is primarily the web application on port 80.

---

# 2. Investigating the Web Application

Navigate to:

```
http://cryptofailure.thm/
```

The page simply reports that we're logged in.

At first this doesn't look particularly interesting.

So intercept the request with **Burp Suite**.

The application creates two cookies:

```
secure_cookie
user
```

The `user` cookie identifies the current user, while `secure_cookie` supposedly proves that the user is legitimate.

The response also contains an interesting HTML comment:

```
<!-- TODO: remember to remove .bak files -->
```

That is a huge clue.

The developer apparently left backup files on the server.

---

# 3. Finding the Backup Source Code

We can search for backup files with Gobuster:

```
gobuster dir -u http://cryptofailure.thm/ \
-w /usr/share/wordlists/dirb/common.txt \
-x bak
```

Among the results is:

```
/index.php.bak
```

Download it:

```
wget http://cryptofailure.thm/index.php.bak
```

Now we can examine the application's source code.

This is the point where the room becomes much more interesting.

---

# 4. Understanding the Authentication Scheme

The application imports:

```
include('config.php');
```

The important variable is:

```
$ENC_SECRET_KEY
```

We don't know its value yet.

The application generates a string resembling:

```
user:USER_AGENT:ENC_SECRET_KEY
```

For example:

```
guest:Mozilla/5.0 (...):<SECRET>
```

It then passes this string to:

```
make_secure_cookie()
```

The important part of that function is:

```
foreach (str_split($text,8) as $el) {
    $secure_cookie .= cryptstring($el,$SALT);
}
```

So the string is divided into **8-character chunks**.

Each chunk is passed to:

```
cryptstring()
```

which simply calls:

```
crypt($what,$SALT);
```

Therefore the application isn't using some sophisticated custom encryption algorithm.

It is repeatedly applying PHP's `crypt()` to individual 8-byte chunks using the same salt.

That's the fundamental weakness.

---

# 5. Why the Cookie Can Be Forged

The verification function does something particularly dangerous:

```
$salt = substr($_COOKIE['secure_cookie'],0,2);
```

The salt is taken directly from the attacker's supplied cookie.

More importantly, the application reconstructs the plaintext using:

```
$user + ":" + User-Agent + ":" + ENC_SECRET_KEY
```

Both `user` and `User-Agent` are controllable by us.

The secret key isn't — but it doesn't appear until later in the string.

This gives us a **known-plaintext / controllable-input situation**.

Independent walkthroughs confirm that the vulnerability comes from the 8-byte chunking combined with the attacker-controlled `User-Agent` and weak `crypt()` construction.

---

# 6. Getting Administrator Access

Suppose our original user is:

```
guest
```

and the beginning of the User-Agent is:

```
Mo
```

The first 8-byte chunk therefore becomes:

```
guest:Mo
```

The corresponding hash is:

```
crypt("guest:Mo", "wO")
```

But we want:

```
admin
```

So the first chunk should become:

```
admin:Mo
```

We can calculate its hash using PHP:

```
php -a
```

Then:

```
echo crypt("admin:Mo", "wO");
```

The important point is that we don't need to know the secret key to replace the **first chunk**.

We can construct a cookie whose first hash corresponds to:

```
admin:Mo
```

while keeping the remaining hashes from the legitimate cookie.

Then set:

```
user=admin
```

and replace the first chunk of `secure_cookie`.

Reload the page.

The application now accepts us as `admin`.

### Web flag

The first flag is:

```
THM{ok_you_f0und_w3b_fl4g_6cbe2bc}
```

This matches independent walkthroughs of the room.

---

# 7. The Second Challenge — Recover `ENC_SECRET_KEY`

After obtaining the web flag, the application tells us:

```
Now I want the key.
```

So we need to recover:

```
ENC_SECRET_KEY
```

A naive approach would be extremely inefficient.

Each chunk contains 8 characters, so brute-forcing an entire unknown 8-character chunk would require an enormous search space.

Instead, we exploit the application's ability to control the **User-Agent length**.

---

# 8. The Alignment Attack

The important structure is:

```
guest:<USER_AGENT>:<SECRET_KEY>
```

The secret begins immediately after the colon following the User-Agent.

Because the application hashes the string in 8-byte blocks, we can manipulate the User-Agent so that:

```
7 known characters + 1 unknown secret character
```

occupy an entire block.

For example, we can arrange a block like:

```
AAAAAAA?
```

where `?` is the next unknown character of the secret.

Now we don't have to brute-force eight characters.

We only need to test possible values for **one character**.

That's the key insight.

Independent solutions describe the same technique: use controlled User-Agent padding to align the unknown secret character into an 8-byte `crypt()` block, then compare the resulting hash with the corresponding block in `secure_cookie`.

---

# 9. Recovering the Key One Character at a Time

The process is:

### Step 1

Choose a User-Agent length that aligns the next unknown character.

### Step 2

Request a fresh cookie.

### Step 3

Extract the relevant 13-character `crypt()` output.

### Step 4

Try every possible character.

For each candidate:

```
known_7_bytes + candidate
```

calculate:

```
crypt(candidate_block, salt)
```

### Step 5

Compare the result against the corresponding hash in `secure_cookie`.

If they match:

```
candidate = correct character
```

Then move to the next character.

This is effectively a **byte-at-a-time recovery attack**.

---

# 10. Automating the Attack

A Python implementation can automate the alignment and character testing:

```
#!/usr/bin/env python3

import crypt
import requests
import urllib.parse
import string

BASE_URL = "http://cryptofailure.thm/"

USERNAME = "guest:"
SEPARATOR = ":"

CHARSET = string.printable


def get_secure_cookie(user_agent):
    session = requests.Session()

    response = session.get(
        BASE_URL,
        headers={"User-Agent": user_agent}
    )

    cookie = session.cookies.get("secure_cookie")

    return urllib.parse.unquote(cookie)


def main():

    discovered = ""

    while True:

        # Align the next unknown character
        padding_length = (
            7 - len(USERNAME + SEPARATOR + discovered)
        ) % 8

        user_agent = "A" * padding_length

        prefix = (
            USERNAME +
            user_agent +
            SEPARATOR +
            discovered
        )

        block_index = len(prefix) // 8

        secure_cookie = get_secure_cookie(user_agent)

        # Each DES-crypt result occupies 13 characters
        target_block = secure_cookie[
            block_index * 13:
            (block_index + 1) * 13
        ]

        salt = target_block[:2]

        found = False

        for char in CHARSET:

            candidate = (prefix + char)[-8:]

            candidate_hash = crypt.crypt(
                candidate,
                salt
            )

            if candidate_hash == target_block:

                discovered += char

                print(
                    f"[+] Found: {char} "
                    f"-> {discovered}"
                )

                found = True
                break

        if not found:
            break

    print("\n[+] Key:")
    print(discovered)


if __name__ == "__main__":
    main()
```

The exact implementation varies between writeups because the target's generated salt and alignment strategy can differ, but the underlying attack remains the same.

---

# 11. The Encryption Key

The recovered key is:

```
THM{Traditional_Own_Crypto_is_Always_Surprising!_and_this_hopefully_is_not_easy_to_crack_e41d20b5b0989cac65ed4a090cace944bf30e6d3ab88f9d447f52fd2140525b9}
```

This is the second answer required by the TryHackMe room.

TryHackMe currently lists the two questions as **the web flag** and **the encryption key**.

---

# 12. What Actually Went Wrong?

This room is less about "breaking cryptography" and more about **breaking a bad authentication design**.

The application made several serious mistakes:

### 1. Rolling its own cryptographic scheme

Instead of using a proper authenticated session mechanism, it constructed its own cookie format.

### 2. Reusing the same salt

The same salt is used for every 8-byte block.

### 3. Predictable block structure

The plaintext is split into independent 8-byte chunks.

### 4. Attacker-controlled plaintext

We control:

```
user
User-Agent
```

This gives us significant control over the data being hashed.

### 5. Attacker-controlled salt during verification

The verifier obtains the salt from:

```
$_COOKIE['secure_cookie']
```

rather than maintaining a trusted server-side value.

### 6. No proper integrity mechanism

A secure authentication cookie should not simply be a collection of independently hashed pieces.

The application essentially created a homemade authentication protocol — and the structure allowed us to manufacture valid-looking authentication data.

---

# 13. Attack Chain

The whole room can be summarized as:

```
Nmap
  ↓
Port 80
  ↓
Web application
  ↓
Burp Suite
  ↓
Interesting HTML comment
  ↓
index.php.bak
  ↓
Source-code disclosure
  ↓
Understand secure_cookie
  ↓
PHP crypt() + 8-byte chunks
  ↓
Control user + User-Agent
  ↓
Forge first cookie block
  ↓
Become admin
  ↓
WEB FLAG
  ↓
Control User-Agent length
  ↓
Align secret character
  ↓
Brute-force one character
  ↓
Repeat
  ↓
ENC_SECRET_KEY
```

## Flags / Answers

|Question|Answer|
|---|---|
|**Web flag**|`THM{ok_you_f0und_w3b_fl4g_6cbe2bc}`|
|**Encryption key**|`THM{Traditional_Own_Crypto_is_Always_Surprising!_and_this_hopefully_is_not_easy_to_crack_e41d20b5b0989cac65ed4a090cace944bf30e6d3ab88f9d447f52fd2140525b9}`|
