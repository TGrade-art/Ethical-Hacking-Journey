# Grep

**Platform:** TryHackMe  
**Date:** 2026-08-26  
**Difficulty:** Easy  
**Category:** OSINT / Web / Linux  
**Room URL:** [TryHackMe — Grep](https://tryhackme.com/room/grepftp?utm_source=chatgpt.com)  
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

This room was basically a mixture of **recon, OSINT, web enumeration, Burp Suite, and source-code analysis**.

The main idea was that the website had a bunch of information exposed in places it really shouldn't have been. Instead of just blindly attacking the application, I had to piece together information from the website, its API, GitHub commits, uploaded files, and a database backup.

And honestly, the OSINT part was probably my favorite because the website name itself gave me something to search for. 😎

---

# 🔍 Reconnaissance

As always, I started with Nmap.

```bash
nmap -sC -sV -p- 10.129.148.135
```

The important results were:

```text
22/tcp     open  ssh
80/tcp     open  http
443/tcp    open  https
51337/tcp  open  http
```

The target was running **Apache 2.4.41 on Ubuntu**.

Port 80 showed the default Apache page, while 443 returned a `403 Forbidden` and port `51337` returned a `400 Bad Request`.

The interesting part was the hostname revealed by the HTTPS certificate:

```text
grep.thm
```

So I added it to my `/etc/hosts` file.

```bash
sudo nano /etc/hosts
```

And added:

```text
10.129.148.135 grep.thm
```

Now I could properly access:

```text
https://grep.thm
```

---

# 🧭 Steps I Took

## Step 1 — Web Enumeration

First I wanted to see what was hiding on the web server.

```bash
dirb http://10.129.148.135
```

DIRB found some interesting paths, including:

```text
/index.php
/javascript/
/javascript/jquery/gateway
/phpmydmin
/server-status
```

Some of the paths returned `403`, so they weren't immediately accessible, but they still confirmed that additional functionality existed on the server.

At this point I started looking more closely at the actual application instead of just throwing more wordlists at it.

---

# Step 2 — Finding the API Key

The website had a registration function, but when I tried to register, I got an error saying that the API key was invalid or expired.

That was interesting.

So instead of just guessing the key, I intercepted the registration request with **Burp Suite**.

Looking at the request showed that an API key was being sent with the registration request.

Something like:

```text
Api-key: <API_KEY>
```

Now I knew exactly what I needed to find.

### OSINT Time 🔎

The website was called `grep.thm`, and the application appeared to be PHP-based.

So I searched for the application/source code online and found a GitHub repository that looked related to the target.

Looking through the repository and its commit history revealed an older commit containing the API key used by the registration system.

The key was:

```text
ffe60ecaa8bba2f12b43d1a4b15b8f39
```

This answered the first question.

### Why this mattered

This was a great example of why **secrets shouldn't be committed to public repositories**.

Even if a developer deletes a secret from the current version of a project, it can still remain inside the Git history.

---

# Step 3 — Using the API Key

Now that I had the correct API key, I went back to Burp Suite.

I intercepted the registration request and replaced the expired API key with the one I found in the Git history.

Then I forwarded the request.

And...

**BOOM.** 😎

The registration functionality worked.

The first flag was:

```text
THM{4ec9806d7e1350270dc402ba870ccebb}
```

---

# Step 4 — Investigating the File Upload

While going through the application, I remembered that the directory enumeration had revealed:

```text
upload.php
```

So obviously I wanted to see what the upload functionality was doing.

I attempted to upload a PHP file.

The application rejected it.

I tried changing things like the filename and MIME type, but the server was still detecting the file.

So I went back to the source code.

And this is where things got interesting.

The application wasn't simply checking the filename.

It was checking the **hexadecimal/magic bytes of the uploaded file**.

Basically, it was looking at the beginning of the file to determine what type of file it was.

That gave me a much better understanding of what the upload filter was actually doing.

---

# Step 5 — Upload Filter Bypass

Since the filter was checking the file's initial bytes, I modified the beginning of the uploaded file so that it matched one of the file signatures the application accepted.

However, there was a catch.

Changing the first bytes also damaged the beginning of the PHP code.

So the uploaded file existed, but it didn't execute correctly.

I had to adjust the PHP payload so that the PHP opening tag was placed after the bytes/comments being used to satisfy the file-type check.

After making that adjustment, the uploaded file was accepted while preserving the PHP code.

---

# Step 6 — Finding the Uploaded Files

Now I needed to figure out where the server stored uploaded files.

I checked the likely upload directory:

```text
/uploads
```

And found that the uploaded file was there.

Opening it directly didn't immediately execute the PHP code, so I continued investigating how the application handled uploaded files.

This was another good reminder that **uploading a file and getting server-side code execution are two completely different things**.

---

# Step 7 — Finding the Database Backup

While looking through the application files, I found a backup directory.

Inside it was a SQL database backup.

That was VERY interesting. 👀

I inspected the database contents and found information about the application's users.

One of the questions asked:

> What is the email of the "admin" user?

The answer was:

```text
admin@searchme2023cms.grep.thm
```

So now I had an actual administrator email address.

---

# Step 8 — Finding LeakChecker

While continuing to inspect the application files, I also found a reference to:

```text
leakchecker
```

Since the target already had an unusual HTTP service on port `51337`, I wondered if this was another virtual host.

So I added another hostname to `/etc/hosts`:

```text
10.129.148.135 leakchecker.grep.thm
```

Then I accessed the application on port `51337`.

Sure enough, I found the **LeakChecker** application.

The room asked:

> What is the host name of the web application that allows a user to check an email for a possible password leak?

The answer was:

```text
leakchecker.grep.thm
```

---

# Step 9 — Finding the Admin Password

The LeakChecker application and the information found earlier gave me another lead.

By following the exposed information from the application and its source, I was able to identify the administrator's password:

```text
admin_tryhackme!
```

So at this point I had:

```text
Username: admin
Password: admin_tryhackme!
```

I tried logging into the application with those credentials.

The login functionality itself wasn't actually implemented properly, which was honestly pretty funny. 😂

Still, the credentials answered the final question and demonstrated how dangerous it is to expose passwords and sensitive application information through source code, backups, or related services.

---

# 🛠 Tools & Commands Used

|Tool / Command|What I Used It For|
|---|---|
|`nmap -sC -sV -p-`|Full port and service enumeration|
|`dirb`|Web directory enumeration|
|Browser|Inspecting the web application|
|Burp Suite|Intercepting and modifying HTTP requests|
|GitHub|OSINT and source-code/commit-history investigation|
|`/etc/hosts`|Mapping discovered hostnames to the target|
|SQL backup|Investigating exposed application data|

---

# 🚩 Answers / Flags Found

|Question|Answer|
|---|---|
|API key|`ffe60ecaa8bba2f12b43d1a4b15b8f39`|
|First flag|`THM{4ec9806d7e1350270dc402ba870ccebb}`|
|Admin email|`admin@searchme2023cms.grep.thm`|
|LeakChecker hostname|`leakchecker.grep.thm`|
|Admin password|`admin_tryhackme!`|

---

# 💡 What I Learned

- **Git history is dangerous when developers commit secrets.** Deleting a password or API key from the current source doesn't necessarily remove it from the repository's history.
    
- **OSINT can directly help with penetration testing.** Searching for the application's name led me to source code that revealed information the live application wasn't giving me.
    
- **Burp Suite is extremely useful for understanding web applications.** Instead of only interacting with forms normally, I could actually see and modify the HTTP requests being sent.
    
- **File upload filters aren't automatically secure.** A filter based on filenames, MIME types, or magic bytes can have weaknesses if it isn't designed properly.
    
- **Backups are part of the attack surface.** Finding a database backup containing user information was a huge information leak.
    
- **Subdomains can completely change the attack surface.** `grep.thm` wasn't the only interesting application; discovering `leakchecker.grep.thm` exposed another service.
    

---

# ❓ What Confused Me / What I Want to Research Next

The biggest thing I want to understand better is **how file-type detection actually works**.

I understood that the application was checking hexadecimal/magic-byte information, but I'd like to learn more about:

- File signatures and magic bytes
    
- How servers determine MIME types
    
- Why changing a file's first bytes can affect how an application treats it
    
- How secure upload systems should validate files
    
- How Git history can accidentally expose secrets
    

I'd also like to get better at **OSINT during CTFs**, because this room showed me that sometimes the information you need isn't on the target machine at all.

---

# 🔗 Linked Notes

- [[Nmap]]
    
- [[Web Enumeration]]
    
- [[Burp Suite]]
    
- [[OSINT]]
    
- [[Git]]
    
- [[Git History]]
    
- [[File Upload Vulnerabilities]]
    
- [[Virtual Hosts]]
    
- [[SQL Databases]]
    

---

## 🏁 Final Thoughts

This room was actually really fun because it wasn't just:

> **Nmap → exploit → flag.**

It was more like:

**Recon → hostname → web app → Burp → OSINT → Git history → API key → source code → upload functionality → database backup → subdomain → leaked credentials.**

Everything connected to something else.

And that's probably the biggest thing I took away from **Grep**: **don't tunnel-vision on one attack path.** If something doesn't work, look at what information the application is accidentally giving you and follow the trail. 😎

_Report written in [[TryHackMe Vault]] — part of my ethical hacking journey 🛡️_