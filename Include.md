# TryHackMe Include — Write-Up

Before starting the lab, I added the target IP to `/etc/hosts` and mapped it to:

```text
include.thm
```

Target:

```text
http://include.thm
```

The room also had a few questions to answer along the way, so let's get started.

## 1. Enumeration

As usual, I started with a port scan. I normally use **RustScan** for this because it is fast and gives me a good idea of what is exposed before I start digging deeper.

The scan showed quite a few open ports. A lot of them were related to mail services such as **IMAP** and **POP3**.

I checked them briefly, but nothing immediately stood out as exploitable, so I decided to focus on the web applications instead.

One of the more interesting ports was **50000**, which was running a **SysMon** web application.

![SysMon]

The page had a login form called **"Restricted Portal."**

I tried the usual SQL injection tests, but the login wasn't vulnerable.

I also checked the page source for anything interesting, but there were no obvious hints there either.

Moving on.

Another interesting port was **4000**, which was running a **Node.js** application.

![Node.js application]

And luckily, the creator basically left me a login hint:

```text
guest : guest
```

So I logged in.

After getting access to the application, I started fuzzing the web server on port 4000 to look for hidden directories and files.

I didn't find anything particularly useful from the directory fuzzing, though.

At this point, the interesting part of the challenge was inside the authenticated application.

# 2. Privilege Escalation via BOPLA

While looking through the account information page, I noticed that I could add new objects to my profile.

That by itself wasn't too interesting.

What caught my attention was that I could also **modify properties that I shouldn't have been able to modify**.

So I started looking at the request more closely.

One of the properties was:

```json
"isAdmin": false
```

I thought:

> What happens if I simply change this to `true`?

And... it worked.

```json
"isAdmin": true
```

This was a classic **BOPLA — Broken Object Property Level Authorization** vulnerability.

Basically, the application trusted the client to modify properties that should have been controlled by the server.

After changing the value, I suddenly had access to additional administrator functionality.

And yep — **new tabs appeared.**

## API

The first place I checked was the **API** tab.

The API contained credentials that I needed to access the SysMon application.

The problem was that the API was only accessible internally, so I needed a way to make the server request it for me.

That's where the next vulnerability came in.

## SSRF

While going through the different tabs, I found an input field in the settings page that accepted a URL.

That immediately made me think:

**SSRF.**

I intercepted the request with Burp Suite and looked at the parameters.

The interesting parameter was:

```text
url
```

So instead of giving it an external URL, I changed it to point to the internal API through localhost.

For example:

```text
http://127.0.0.1/...
```

The idea was simple:

```text
My request
     ↓
include.thm
     ↓
Server makes request
     ↓
Internal API on localhost
     ↓
Sensitive information
```

The application then returned the response through the `/admin/settings` page.

The response was **Base64 encoded**, so I decoded it and got the information I was looking for.

After decoding the response, I finally had the credentials for the SysMon application:

```text
administrator : S$9$qk6d#**LQU
```

Now I had everything I needed.

I went back to the SysMon application on port `50000` and logged in as the administrator.

And there it was:

**the first flag.**

---

# 3. Getting the Second Flag

Getting the second flag had **two different methods**.

Both methods started with the same thing:

**finding a Local File Inclusion vulnerability.**

## 3.1 LFI → RCE

While looking around the SysMon application, I started checking the page source.

One page immediately caught my attention:

```text
dashboard.php
```

I also found this interesting parameter:

```text
profile.php?img=profile.png
```

The application was taking a filename from the `img` parameter.

That made me think:

> If I can control which image gets loaded, can I make it load something else?

So I started testing for path traversal.

I tried a payload like:

```text
....//....//....//....//....//....//....//....//....//....//etc/passwd
```

And it worked.

I could read `/etc/passwd`.

The file showed two interesting users:

```text
joshua
charles
```

I initially tried going down the SSH route and testing whether I could get access using those accounts.

But there was another, more interesting way to turn the LFI into code execution.

## 3.2 LFI → RCE via Mail Log Poisoning

The second method was **mail log poisoning**.

The basic idea is:

1. Find an LFI vulnerability.
    
2. Find a log file that contains user-controlled input.
    
3. Inject PHP code into that log.
    
4. Use the LFI to include the poisoned log.
    
5. The PHP code gets executed.
    

Since I already had LFI, I checked whether I could read the mail log:

```text
/var/log/mail.log
```

Using the same path traversal technique, I was able to access it.

Now I needed to get some PHP code into the log.

I sent a request containing a basic PHP payload so that it would be written into the mail log.

The server returned a `501` response, but that wasn't really a problem.

The important part was that the request had already been **logged**.

So now I had:

```text
Malicious input
      ↓
/var/log/mail.log
      ↓
LFI includes the log
      ↓
PHP gets interpreted
      ↓
Command execution
```

I then used the LFI to include the poisoned log and passed a command through the `cmd` parameter.

For example:

```text
&cmd=id
```

And I got command execution.

I could then run commands such as:

```text
ls -la /var/www/html
```

At this point, the LFI had effectively become **RCE**.

That gave me the access I needed to finish the room and retrieve the second flag.

---

# 4. Prevention

Now let's look at how these vulnerabilities could have been prevented.

## 4.1 Broken Object Property Level Authorization (BOPLA)

The main problem was that the server trusted the client to modify properties it shouldn't have been allowed to touch.

A few ways to prevent this:

- Don't blindly serialize entire objects with methods such as `to_string()` or `to_json()`.
    
- Explicitly define which properties can be returned to the client.
    
- Use schema-based validation for API requests and responses.
    
- Only allow users to modify properties they are actually authorized to change.
    
- Keep sensitive properties such as `isAdmin` under server-side control.
    

In this case, the application should never have accepted:

```json
"isAdmin": true
```

from a normal user.

---

## 4.2 Server-Side Request Forgery (SSRF)

The SSRF happened because the application allowed me to control the URL that the server requested.

To prevent this:

- Validate user-supplied URLs.
    
- Use an allowlist of trusted domains.
    
- Block requests to localhost and internal IP ranges.
    
- Resolve the hostname and verify that it doesn't point to an internal address.
    
- Be careful with redirects and DNS rebinding.
    
- Isolate outbound requests through a controlled proxy when possible.
    

The important thing is that the server shouldn't blindly trust a URL just because it came from the application itself.

---

## 4.3 Local File Inclusion (LFI)

The LFI existed because user input was being used directly when loading a file.

A safer approach is to:

- Use an allowlist of permitted files.
    
- Don't allow arbitrary paths from users.
    
- Canonicalize paths before accessing them.
    
- Make sure the final path stays inside the intended directory.
    
- Reject traversal sequences such as `../`.
    

For example, instead of allowing:

```text
?img=<anything>
```

the application could map an ID to a predefined filename.

That way, the user never gets direct control over the filesystem path.

---

## 4.4 Mail Log Poisoning

Finally, mail log poisoning can be prevented by making sure user-controlled data cannot become executable code.

Some useful protections include:

- Sanitize and validate user-controlled email data.
    
- Properly escape special characters.
    
- Don't allow sensitive log files to be accessed through user-controlled file paths.
    
- Use structured logging where possible.
    
- Restrict access to log files.
    
- Never execute or include log files as application code.
    

The biggest issue here was the combination of **LFI + user-controlled log data**.

Individually, these might not have been enough to get RCE, but together they turned a simple file-reading vulnerability into full command execution.

# Conclusion

This was a pretty nice chain because it wasn't just one vulnerability.

The attack path was basically:

```text
Enumeration
    ↓
Node.js application
    ↓
BOPLA
    ↓
isAdmin = true
    ↓
Admin functionality
    ↓
SSRF
    ↓
Internal API
    ↓
SysMon credentials
    ↓
Administrator access
    ↓
LFI
    ↓
Mail Log Poisoning
    ↓
RCE
    ↓
Second Flag
```

The main things I took from this room were **BOPLA, SSRF, LFI, and log poisoning**.

The interesting part was seeing how multiple small weaknesses could be chained together to completely compromise the application.