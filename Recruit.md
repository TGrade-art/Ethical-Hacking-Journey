# Recruit

**Platform:** TryHackMe
**Date:** 2026-10-19
**Difficulty:** Medium
**Category:** Web Security / Local File Inclusion / SQL Injection
**Room URL:** [https://tryhackme.com/room/recruit](https://tryhackme.com/room/recruit)
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

Recruit is a web application security room focused on finding vulnerabilities in a recruitment portal and using them to gain access to different levels of the application.

I started with basic reconnaissance, discovered an exposed file inclusion vulnerability that leaked HR credentials, and used those credentials to log in as a normal user. From there, I found an SQL injection vulnerability in the search functionality and used SQLMap to enumerate the database and retrieve the administrator credentials.

The main goal was to go from an unauthenticated user to a normal account, then escalate to administrator access.

---

## 🔍 Reconnaissance

As usual, I started with Nmap to identify the open ports and services running on the target. Before doing anything else, I wanted to understand what I was dealing with.

### Nmap Scan

**Command used:**

```
nmap -sS -vv -p- -A recruit.thm
```

I used `-sS` for a SYN scan, `-vv` for more detailed output, `-p-` to scan all TCP ports, and `-A` to enable service detection, OS detection, default scripts, and traceroute.

**Results:**

|Port|Service|Version|Notes|
|---|---|---|---|
|22/tcp|SSH|OpenSSH 8.2p1|Remote login service|
|53/tcp|DNS|ISC BIND 9.16.1|DNS service|
|80/tcp|HTTP|Apache 2.4.41|Recruit web application|

The web server was running Apache on port 80, so that was my main focus. SSH and DNS were worth recording, but the website looked like the most promising starting point.

### Setting Up the Hostname

The target was using the hostname `recruit.thm`, so I added the target IP and hostname to my local hosts file.

```
sudo nano /etc/hosts
```

I added:

```
<TARGET_IP> recruit.thm
```

I replaced `<TARGET_IP>` with the actual IP assigned by TryHackMe.

This allowed me to access the application using `http://recruit.thm` instead of repeatedly typing the IP address.

### Directory Enumeration

I opened the website and found a login page. I tried a few common credentials and checked for obvious SQL injection issues, but nothing immediately worked.

Rather than wasting time guessing passwords, I moved on to directory enumeration with Gobuster.

**Command used:**

```
gobuster dir \
  -u http://recruit.thm \
  -w /usr/share/wordlists/dirb/common.txt
```

The original walkthrough used `directory-list-2.3-medium.txt`; either wordlist can be used, but the results may differ.

**What I found:**

- `/mail`
    
- `/assets`
    
- `/phpmyadmin`
    
- `/server-status`
    

The `/mail` directory caught my attention because it appeared to contain information about how the application handled its email and configuration.

I decided to investigate that before going back to the login page.

---

## 🧭 Steps I Took

### Step 1 — Investigating the Mail Directory

- **What I did:**
    
    I browsed the `/mail` directory and inspected the information exposed by the application.
    
- **What I found:**
    
    The page gave me a clue that the application used a `config.php` file and that administrator credentials were stored in the database.
    
- **Why it matters:**
    
    Configuration files can contain database connection details, usernames, passwords, and other information that should never be exposed through a public web application.
    
    I didn't have the administrator password yet, but this finding gave me two useful directions to investigate: the configuration file and the database.
    

### Step 2 — Finding the File Inclusion Vulnerability

- **What I did:**
    
    I continued exploring the application's file-related functionality and found an endpoint that accepted a `cv` parameter.
    
    The endpoint appeared to load a file based on the supplied parameter. I tested whether I could make it load a local file instead of the intended resource.
    
- **Command/Tool used:**
    
    I tested the endpoint in the browser, using a request in this general format:
    
    ```
    http://recruit.thm/file.php?cv=config.php
    ```
    
    The application indicated that local files were allowed. That was an important clue because it suggested that the parameter might not be restricted to the expected file locations.
    
    I then tested a PHP-supported stream wrapper:
    
    ```
    http://recruit.thm/file.php?cv=file://config.php
    ```
    
- **What I found:**
    
    The application exposed information that led me to the HR account credentials:
    
    ```
    Username: hr
    Password: hrpassword123
    ```
    
- **Why it matters:**
    
    The application was allowing user-controlled input to influence which file it accessed. If file paths or stream wrappers are not properly restricted, this can lead to Local File Inclusion (LFI) or other file disclosure vulnerabilities.
    
    The important lesson here is that I didn't need to guess the password or break the login form. I found credentials through a weakness in how the application handled files.
    
    **Technical note:** The exact behavior of PHP stream wrappers depends on the application's implementation and PHP configuration. The `file://` wrapper does not automatically bypass every filter, so this result should be verified against the actual room instance.
    

### Step 3 — Logging In as a Normal User

- **What I did:**
    
    With the HR credentials recovered, I returned to the main login page and authenticated.
    
- **Credentials used:**
    
    ```
    Username: hr
    Password: hrpassword123
    ```
    
- **What I found:**
    
    I successfully logged in as a normal user and obtained the first flag:
    
    ```
    THM{LOGGED_IN_USER}
    ```
    
- **Why it matters:**
    
    This confirmed that the credentials I had discovered were valid and that the file inclusion issue had given me a working initial foothold in the application.
    
    At this point, I had completed the first part of the challenge. However, the mail directory had already suggested that administrator credentials were stored in the database, so I still had another lead to investigate.
    

### Step 4 — Testing the Search Functionality for SQL Injection

- **What I did:**
    
    After logging in, I investigated the application's search functionality. I wanted to determine whether the search parameter was safely handled by the backend.
    
    I submitted a single quote (`'`) as a basic initial test and observed the application's response.
    
- **Tool used:** Browser and Burp Suite
    
- **What I found:**
    
    The response indicated that the search parameter was vulnerable to SQL injection.
    
    Instead of stopping there, I captured the relevant HTTP request in Burp Suite and saved it to a file named `request.txt`.
    
- **Why it matters:**
    
    SQL injection happens when an application incorporates untrusted input into a database query without handling it safely. Depending on the query and database permissions, this can expose records that the current user should not be able to access.
    
    I now had a potential path from a normal user account to the administrator account.
    

### Step 5 — Enumerating the Databases with SQLMap

- **What I did:**
    
    I used SQLMap to test the captured request and identify the databases accessible through the vulnerable parameter.
    
- **Command used:**
    
    ```
    sqlmap -r request.txt -p search --dbs
    ```
    
    The options mean:
    
    - `-r request.txt` — load the saved HTTP request.
        
    - `-p search` — focus testing on the `search` parameter.
        
    - `--dbs` — enumerate available databases.
        
- **What I found:**
    
    SQLMap identified these databases:
    
    ```
    information_schema
    mysql
    performance_schema
    phpmyadmin
    recruit_db
    sys
    ```
    
    The database I was interested in was `recruit_db`, since it appeared to belong to the recruitment application.
    
- **Why it matters:**
    
    Database enumeration helped me move from simply knowing that SQL injection existed to identifying where the application's user information was stored.
    
    I kept the testing focused on the relevant database instead of blindly dumping everything.
    

### Step 6 — Enumerating the Tables

- **What I did:**
    
    Next, I enumerated the tables inside `recruit_db`.
    
- **Command used:**
    
    ```
    sqlmap -r request.txt \
      -p search \
      -D recruit_db \
      --tables
    ```
    
- **What I found:**
    
    The database contained two tables:
    
    ```
    candidates
    users
    ```
    
    The `users` table was the obvious next place to investigate because it was likely to contain account information.
    
- **Why it matters:**
    
    Knowing the database name wasn't enough. I needed to identify which table contained the records relevant to the login system.
    
    This step narrowed down the investigation and gave me a more specific target.
    

### Step 7 — Inspecting the Users Table

- **What I did:**
    
    I checked the columns in the `users` table to understand its structure.
    
- **Command used:**
    
    ```
    sqlmap -r request.txt \
      -p search \
      -D recruit_db \
      -T users \
      --columns
    ```
    
- **What I found:**
    
    SQLMap identified three columns in the table.
    
    The original walkthrough did not list their names individually, so I'm not going to invent them here. The next step was to retrieve the relevant user records.
    
- **Why it matters:**
    
    Enumerating columns helps establish what information a table stores before extracting records. In this case, the table was directly relevant to the application's authentication system.
    

### Step 8 — Retrieving the Administrator Credentials

- **What I did:**
    
    I used SQLMap to dump the application's database and inspected the results for the administrator account.
    
- **Command used in the original walkthrough:**
    
    ```
    sqlmap -r request.txt -D recruit_db --dump-all
    ```
    
    This command attempts to dump all tables in the selected database. For a more focused test, I would target only the `users` table:
    
    ```
    sqlmap -r request.txt \
      -p search \
      -D recruit_db \
      -T users \
      --dump
    ```
    
- **What I found:**
    
    The administrator credentials were:
    
    ```
    Username: admin
    Password: admin@001admin
    ```
    
- **Why it matters:**
    
    The SQL injection had exposed credentials belonging to a higher-privileged account. That meant the application had failed to properly separate the information available to a normal user from sensitive authentication data.
    
    This was the key escalation step: I had gone from an unauthenticated visitor to HR access, and then used the SQL injection vulnerability to obtain administrator credentials.
    
    In a real application, passwords should be stored using an appropriate password-hashing scheme, and database queries should use parameterized statements rather than string concatenation.
    

### Step 9 — Logging In as Administrator

- **What I did:**
    
    I returned to the login page and used the administrator credentials recovered from the database.
    
- **Credentials used:**
    
    ```
    Username: admin
    Password: admin@001admin
    ```
    
- **What I found:**
    
    I successfully authenticated as the administrator and obtained the second flag:
    
    ```
    THM{LOGGED_IN_ADM1N1}
    ```
    
- **Why it matters:**
    
    This completed the room. The administrator account was not compromised through password guessing; it was exposed because the application allowed database extraction through SQL injection.
    

---

## 🛠 Commands Used

|Command|What It Does|
|---|---|
|`nmap -sS -vv -p- -A recruit.thm`|Scans TCP ports and identifies services and versions.|
|`sudo nano /etc/hosts`|Adds the target hostname to the local hosts file.|
|`gobuster dir -u http://recruit.thm -w /usr/share/wordlists/dirb/common.txt`|Searches for discoverable web directories.|
|`sqlmap -r request.txt -p search --dbs`|Enumerates databases through the vulnerable request parameter.|
|`sqlmap -r request.txt -p search -D recruit_db --tables`|Lists tables in the target database.|
|`sqlmap -r request.txt -p search -D recruit_db -T users --columns`|Enumerates the columns in the users table.|
|`sqlmap -r request.txt -p search -D recruit_db -T users --dump`|Retrieves records from the users table.|

---

## 🚩 Flags Found

|Access Level|Flag|
|---|---|
|Normal user|`THM{LOGGED_IN_USER}`|
|Administrator|`THM{LOGGED_IN_ADM1N1}`|

---

## 📎 Resources Used

- [TryHackMe — Recruit](https://tryhackme.com/room/recruit)
    
- [SQLMap Documentation](https://sqlmap.org/)
    
- [OWASP — SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
    
- [OWASP — Path Traversal](https://owasp.org/www-community/attacks/Path_Traversal)
    

---