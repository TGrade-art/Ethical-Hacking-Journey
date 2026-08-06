
---

# The Hollow Shell

**Platform:** TryHackMe **Date:** 2026-08-06 **Difficulty:** Medium **Category:** Web / Linux / Zip Slip **Room URL:** https://tryhackme.com/room/thehollowshell **Status:** ✅ Completed

---

## 📌 What Was This Room About?

> This room was about exploiting a Zip Slip vulnerability by hiding a malicious reverse shell inside a zip file using path traversal. The goal was to bypass the app's file extension filters, drop a payload into a hidden server directory, and catch a shell when a background cron job executed it automatically.

---

## 🔍 Reconnaissance

### Nmap Scan

```bash
nmap -sV -sC -p- 10.128.181.126
```

**Results:**

|Port|Service|Version|Notes|
|---|---|---|---|
|5000|http|Python/Flask|Hosting the theme deployment web application|

---

## 🧭 Steps I Took

### Step 1 — Poking Around the Web Interface

- **What I did:** I opened up the web dashboard on port 5000 to see what I was working with.
- **Command/Tool used:** Web browser navigating to `http://10.128.181.126:5000`
- **What I found:** A file uploader called "Bring a shell ashore" that accepts `.zip` theme bundles and extracts them into a public directory at `/shells/<random_hex_id>/`. It reads a config file called `shell.json` inside the archive, and only allows `png`, `jpg`, `gif`, `svg`, `css`, and `json` extensions.
- **Why it matters:** The extension whitelist looked strict at first glance, but any time a server unzips files without sanitizing the paths inside the archive, things can get interesting 😏

### Step 2 — Testing the Filter and Finding the Crack

- **What I did:** I tried uploading `.php` and `.py` files directly — both got rejected immediately. So I shifted my thinking. If I can't bypass the extension check, maybe I can bypass _where_ the file gets extracted to.
- **Command/Tool used:** Local file testing and directory enumeration.
- **What I found:** Direct browsing to `/shells/` returned 404s because directory listing was disabled. BUT — the unzip engine happily accepted `../../` path traversal operators embedded inside the zip file's internal metadata. Permission denied on the front door, but the back window was wide open 😈
- **Why it matters:** This means I could force the server to drop files outside the public `/shells/` directory entirely and write directly into hidden core directories on the server.

### Step 3 — Building the Zip Slip Payload

- **What I did:** I wrote a Python script to craft a zip file where the internal file path pointed backward into the server's `hooks/` directory — the folder a background cron job watches for scripts to execute.
- **Command/Tool used:**

```bash
# python3 build_exploit.py
```

- **What I found:** Regular Linux zip tools flatten paths and ruin the traversal. Python's `zipfile` library keeps the raw path string intact, so `../../hooks/callback.py` went straight into the archive headers exactly as written.
- **Why it matters:** The server runs a cron job that automatically executes anything dropped into `hooks/`. So once my reverse shell landed there, all I had to do was wait for the cron timer to fire. 🕐

### Step 4 — Starting the Listener and Catching the Shell

- **What I did:** One thing that caught me off guard — the AttackBox uses `ens5` instead of the usual `eth0`, so I had to check my actual IP before setting up the listener. Classic gotcha 😅
- **Command/Tool used:**

```bash
# ip addr show ens5
# nc -lvnp 4444
```

- **What I found:** I uploaded `reverse_shell.zip`, the app accepted it without complaint, and within about 10 seconds the cron job fired, executed `callback.py`, and a shell popped right into my listener. BOYAH!!! 🎉
- **Why it matters:** No firewall drama, no extension block — the background execution job did all the work for me.

---

## 🛠 Commands Used

|Command|What It Does|
|---|---|
|`nmap -sV -sC -p- 10.128.181.126`|Full port scan with service and script detection|
|`ip addr show ens5`|Finds the correct IP address on the AttackBox interface|
|`nc -lvnp 4444`|Starts a listener to catch the incoming reverse shell|
|`python3 build_exploit.py`|Runs the custom script that builds the malicious zip file|
|`cat /home/roomservice/user.txt`|Reads the user flag|

---

## 🚩 Flags Found

| Flag      | Value         |
| --------- | ------------- |
| User flag | THM{REDACTED} |
| Root flag | THM{REDACTED} |

---

## 💡 What I Learned

- **Check your network interface first:** Don't assume `eth0` or `tun0` — on the AttackBox it's `ens5`. A broken reverse shell route is an embarrassing way to waste twenty minutes.
- **Extension filters only protect what they can see:** Filtering filenames at the surface means nothing if the extraction engine trusts the path metadata inside the archive.
- **Background workers are prime targets:** Any directory that a cron job or automated task watches is a potential execution vector if you can write to it.

---

## ❓ What Confused Me / What to Research Next

- Want to understand why standard Linux `zip` flattens traversal paths but Python's `zipfile` library preserves them — there's something interesting in that architectural difference worth digging into.
- Need to research how to properly detect and block Zip Slip vulnerabilities on the defensive side.

---

## 📎 Resources Used

- YouTube Walkthrough — TryHackMe: The Hollow Shell Room
- [TryHackMe Hacktivities](https://tryhackme.com/hacktivities/search)

---
