# Hammer

**Platform:** TryHackMe  
**Room:** Hammer  
**Room URL:** [https://tryhackme.com/room/hammer](https://tryhackme.com/room/hammer?utm_source=chatgpt.com)  
**Difficulty:** Medium  
**Status:** ✅ Completed

---

## 🧭 Introduction

This room was mainly about web enumeration, finding hidden directories, abusing a weak password-reset system, and then messing with a JWT to get admin access.

The attack chain was basically:

**Nmap → Web Enumeration → Hidden Directory → Password Reset → Recovery Code Brute Force → JWT Analysis → Admin Access**

So let's get into it.

---

# 🔍 1. Reconnaissance

First, as usual, I started with an Nmap scan.

```bash
nmap -sC -sV -p- -T4 -oN nmap.txt 10.10.193.250
```

The important options here are:

- `-sC` — runs Nmap's default scripts.
    
- `-sV` — detects service versions.
    
- `-p-` — scans all 65,535 ports.
    
- `-T4` — speeds up the scan.
    
- `-oN` — saves the results to a file.
    

The scan showed **2 open ports**:

```text
22/tcp
1337/tcp
```

SSH was running on port 22, so I checked out the web server on **1337** first.

---

# 🌐 2. Checking Port 1337

Opening the website gave me a login page.

I tried a few admin email addresses, but none of them worked.

Instead of continuing to guess, I checked the page source.

And that's where things got interesting.

I found a comment saying that the directory names had to start with:

```text
hmr_
```

That gave me something specific to look for.

---

# 🔎 3. Finding Hidden Directories

I used `ffuf` to fuzz for directories beginning with `hmr_`.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
-u http://10.10.193.250:1337/hmr_FUZZ
```

This found:

```text
hmr_logs
```

So obviously, I went to it.

Inside the directory, I found:

```text
/error.logs
```

I checked the log file and found an email address:

```text
tester@hammer.thm
```

Now I had an actual username/email to work with.

---

# 🔑 4. Password Reset

I tried the email address on the password-reset page.

The application sent a **4-digit recovery code**.

The problem was that the code was only valid for a limited amount of time.

Since there are only:

```text
0000 - 9999
```

possible 4-digit combinations, I decided to brute-force the code.

I generated a wordlist containing all of the possible numbers.

I also used Burp Suite to capture the request and obtain the `PHPSESSID` cookie.

---

# 💥 5. Brute-Forcing the Recovery Code

I used `ffuf` to send the recovery codes to the password-reset endpoint.

```bash
ffuf -w numbers.txt \
-u "http://10.10.67.164:1337/reset_password.php" \
-X "POST" \
-d "recovery_code=FUZZ&s=60" \
-H "Cookie: PHPSESSID=3u2ms637t9s7t7nr8p3u220u1k" \
-H "X-Forwarded-For: FUZZ" \
-H "Content-Type: application/x-www-form-urlencoded" \
-fr "Invalid" \
-s
```

Here's what the important options are doing:

- `-X POST` — sends POST requests.
    
- `-d` — specifies the POST data.
    
- `FUZZ` — gets replaced by every value in `numbers.txt`.
    
- `PHPSESSID` — keeps the session active.
    
- `X-Forwarded-For` — changes the apparent client IP for each request, helping bypass the rate limit in this lab.
    
- `Content-Type` — tells the server we're sending form data.
    
- `-fr "Invalid"` — hides responses containing `Invalid`.
    
- `-s` — silent mode.
    

Eventually, I got the recovery code:

```text
2315
```

I submitted it and successfully reset the password.

---

# 🚩 6. Getting the First Flag

After logging in with the new password, I found the first flag.

At this point, I had successfully gone from:

**hidden directory → leaked email → password reset → brute-forced recovery code → account access**

But there was still more to the room.

---

# 🧩 7. Looking at the Dashboard

I used:

```bash
ls
```

to see what files were available.

There was a file that looked interesting, but when I tried using `cat`, it didn't work.

So instead of forcing it, I checked the source code of the page.

And that's where I found something much more interesting:

```text
jwtToken
```

A JWT was being used by the application.

I decoded the token to see what was inside.

---

# 🔐 8. Analyzing the JWT

There were two things that immediately caught my attention:

```text
kid
role
```

### `role`

The JWT contained:

```text
role: user
```

So the application was using the JWT to determine the user's privileges.

That meant changing the role to:

```text
admin
```

could potentially give me administrator access.

### `kid`

The other interesting value was:

```text
kid
```

This tells the application which key should be used to verify the JWT.

And this was especially interesting because I had previously seen a file called:

```text
188ade1.key
```

in the dashboard source.

The problem was that I couldn't read it with `cat`.

So I tried another way.

---

# 📄 9. Reading the Key

I used `curl` to request the file instead.

That allowed me to retrieve the contents of the key.

Now I had the information needed to create a modified JWT.

I changed the JWT so that:

```text
kid → pointed to the key file
role → admin
```

I then used the recovered key as the signing secret for the modified token.

The important lesson here was that the JWT itself wasn't magically secure just because it was signed.

The application was trusting values inside the token, while the key-selection mechanism exposed a way to control which key was used.

---

# 👑 10. Getting Admin Access

After generating the modified JWT, I replaced the original token with my new one.

One important thing I had to remember was to replace the token in **both** locations:

- The `Authorization` header
    
- The cookie
    

After replacing the token and refreshing the page, I now had admin-level access.

And that's where I found the **second flag**.

🏁 **Admin access achieved.**

---

# 🧠 What I Learned

This room had a few really good lessons in it:

- **Don't stop at the obvious attack surface.** The login page looked like the main target, but the page source gave me the `hmr_` clue that led to the logs.
    
- **Information leaks can chain together.** Finding `tester@hammer.thm` eventually led to the password-reset attack.
    
- **Small numeric recovery codes are dangerous when rate limiting can be bypassed.** A 4-digit code only has 10,000 possible combinations.
    
- **Always inspect JWTs.** Things like `role` and `kid` can reveal how an application handles authentication and authorization.
    
- **Source code can reveal things the UI doesn't show.** The key filename and JWT information were both exposed through the application's frontend.
    
- **When one command doesn't work, try another approach.** `cat` didn't work for the key, but `curl` still allowed me to retrieve it.
    

---

# 🎯 Conclusion

Hammer was a pretty good chain of web vulnerabilities because none of the individual steps were ridiculously complicated.

The interesting part was putting everything together.

I started with:

```text
Nmap
 ↓
Port 1337
 ↓
Source code
 ↓
hmr_logs
 ↓
Email address
 ↓
Password reset
 ↓
Recovery code brute force
 ↓
Login
 ↓
JWT
 ↓
kid + exposed key
 ↓
role = admin
 ↓
Admin access
```

The biggest thing I took from this room is that **web enumeration isn't just about finding directories**.

Source code, comments, cookies, logs, JWTs, error messages, and even weird filenames can all become pieces of the attack chain.

And in this room, one small comment in the source code basically pointed me toward the first major foothold.