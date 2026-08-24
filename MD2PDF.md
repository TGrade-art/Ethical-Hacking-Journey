# MD2PDF

**Platform:** TryHackMe  
**Date:** 2026-08-24  
**Difficulty:** Easy  
**Category:** Web / SSRF / PDF / Markdown  
**Room URL:** [TryHackMe MD2PDF](https://tryhackme.com/room/md2pdf?utm_source=chatgpt.com)  
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

This room was about a pretty interesting web application that converts Markdown into PDF files. At first, everything looked completely normal, but after checking how the PDF was generated, I found that it was using **wkhtmltopdf**, which opened the door to an **SSRF vulnerability**.

The goal was basically to use the PDF generator to make the server request something that I couldn't access directly.

---

## 🔍 Reconnaissance

First things first: **Nmap.** 😎

### Nmap Scan

```bash
sudo nmap -sV -Pn <TARGET_IP>
```

**Results:**

|Port|Service|Version|Notes|
|---|---|---|---|
|22|SSH|OpenSSH 8.2p1|SSH service|
|80|HTTP|—|Main MD2PDF application|
|5000|HTTP|—|Another web service|

So we had **three open ports**:

- `22` → SSH
    
- `80` → Main web application
    
- `5000` → Another HTTP service
    

Port 80 was obviously the first place I wanted to investigate.

---

## 🧭 Steps I Took

### Step 1 — Checking the Web Application

- **What I did:**  
    I opened the website running on port 80.
    
- **Command/Tool used:**  
    Web browser.
    
- **What I found:**  
    The website was a Markdown-to-PDF converter. I could enter Markdown into a text box and click **Convert to PDF**.
    
- **Why it matters:**  
    Anything that takes user-controlled input and processes it on the server is worth investigating. I wanted to know exactly what was happening behind the scenes.
    

---

### Step 2 — Checking the Generated PDF

- **What I did:**  
    I entered some Markdown, generated a PDF, downloaded it, and checked its metadata.
    
- **Command/Tool used:**
    

```bash
file document.pdf
exiftool document.pdf
```

- **What I found:**
    

```text
Creator : wkhtmltopdf 0.12.5
Producer: Qt 4.8.7
```

That immediately caught my attention.

- **Why it matters:**  
    The application was using **wkhtmltopdf 0.12.5** to generate the PDFs.
    

I did some research into how this tool handles HTML content and found that it could potentially be abused to make requests to internal resources.

That sounded like **SSRF** territory. 👀

---

### Step 3 — Testing for SSRF

- **What I did:**  
    I tried putting an iframe pointing to localhost inside the Markdown content.
    
- **Payload:**
    

```html
<iframe src="http://localhost/" width="1000" height="2000">
```

- **What I found:**  
    My first attempt didn't give me anything useful.
    
- **Why it matters:**  
    This didn't necessarily mean SSRF wasn't possible. I needed to find something interesting that was only accessible from the server itself.
    

So I went back to enumeration.

---

### Step 4 — Directory Enumeration

- **What I did:**  
    I ran Gobuster against the web server to look for hidden directories.
    
- **Command/Tool used:**
    

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

- **What I found:**  
    I discovered an interesting:
    

```text
/admin
```

Trying to access it normally returned:

```text
403 Forbidden
```

But the response also gave an important clue:

> **This can only be seen internally (localhost:5000).**

And that's when everything started making sense. 😂

---

### Step 5 — Using SSRF to Reach the Internal Admin Page

- **What I did:**  
    The application itself was making requests while generating the PDF, so instead of trying to access `/admin` directly, I made the server request its own internal admin page.
    
- **Payload:**
    

```html
<iframe src="http://localhost:5000/admin" width="1000" height="2000">
```

- **What I found:**  
    I submitted the payload through the Markdown converter and generated the PDF.
    

And...

**BANG. 💥**

The response contained the internal admin page and the flag.

- **Why it matters:**  
    I had successfully used **Server-Side Request Forgery (SSRF)** to access a resource that was restricted to localhost.
    

---

## 🛠 Commands Used

|Command / Tool|What It Does|
|---|---|
|`nmap -sV -Pn <TARGET_IP>`|Finds open ports and identifies services|
|`gobuster dir`|Enumerates hidden directories|
|`file document.pdf`|Identifies the downloaded PDF|
|`exiftool document.pdf`|Extracts PDF metadata|
|Web Browser|Interacts with the MD2PDF application|
|`wkhtmltopdf`|PDF-generation technology identified in the application|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|Room Flag|`THM{...}`|

---

## 💡 What I Learned

- **Always inspect generated files.** The application looked like a simple Markdown converter, but the PDF metadata revealed the exact technology being used: `wkhtmltopdf 0.12.5`.
    
- **SSRF can turn an innocent-looking feature into a security problem.** The PDF generator was processing my input on the server, which meant I could potentially make the server request internal resources for me.
    
- **Internal services aren't automatically safe.** Port `5000` wasn't directly exposed through the application's normal interface, but the server itself could reach it through `localhost`.
    
- **Enumeration matters.** My first SSRF payload didn't work, but instead of giving up, I ran Gobuster and found `/admin`. The `localhost:5000` message was the clue that connected everything together.
    
- **The biggest lesson:** when you find an SSRF, don't just test the main website. Look for **internal services, admin panels, localhost-only endpoints, and other ports** that shouldn't be directly accessible.
    

---

## ❓ What Confused Me / What to Research Next

- My first iframe payload didn't work, so I want to understand more about exactly **how wkhtmltopdf processes external resources** and why certain SSRF payloads work while others don't.
    
- I also want to practice finding SSRF vulnerabilities without relying on a writeup or knowing beforehand that the PDF generator is vulnerable.
    

---

## 🔗 Linked Notes

- [[SSRF]]
    
- [[Web App Enumeration]]
    
- [[Gobuster]]
    
- [[Nmap]]
    
- [[wkhtmltopdf]]
    
- [[Internal Services]]
    
- [[Port Enumeration]]
    

---

## 📎 Resources Used

- [OWASP — Server-Side Request Forgery Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html?utm_source=chatgpt.com)
    

---

_Report written in_ **[[TryHackMe Vault]]** _— part of my ethical hacking journey 🛡️_