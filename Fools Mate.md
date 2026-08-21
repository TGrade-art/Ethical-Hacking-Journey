# Fool's Mate

**Platform:** TryHackMe  
**Date:** 2026-08-21  
**Difficulty:** Easy
**Category:** Web / Client-Side Validation  
**Room URL:** https://tryhackme.com/room/foolsmate
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> This challenge was about exploiting a web application's client-side validation. The game wouldn't let me make the winning chess move, so I had to figure out what was blocking it, find where the validation was happening, and then bypass the browser completely by sending the move directly to the backend.

---

## 🔍 Reconnaissance

### Inspecting the Chess Board

When I opened the challenge, I was given a chess position where White was one move away from checkmate.

The winning move was pretty obvious:

- **Black King:** Trapped on `h8`.
    
- **White Rook:** Sitting on `a1`.
    
- Moving the Rook from `a1` to `a8` gives checkmate.
    

So naturally, I tried it.

The application immediately blocked the move and showed:

> _"I'll shut down your PC if you play that."_

Nice try.

After dismissing the popup, the Rook was reset back to `a1`.

The important part was that the block happened **instantly**. There wasn't any noticeable request being sent to the server first. That made me suspect that the checkmate restriction was being handled by JavaScript running inside the browser.

---

## 🧭 Steps I Took

### Step 1 — Inspecting the Application

- **What I did:** Opened Developer Tools and inspected the HTML and JavaScript loaded by the page.
    
- **Command/Tool used:** Browser Developer Tools
    

The main HTML revealed this script:

```html
<script type="module" src="js/app.js"></script>
```

Since JavaScript loaded by the browser is already being sent to the client, I could download and inspect `app.js`.

---

### Step 2 — Downloading `app.js`

- **What I did:** Used `curl` to retrieve the JavaScript file and searched through it for the logic responsible for blocking my move.
    
- **Command/Tool used:**
    

```bash
curl -s http://<MACHINE_IP>/js/app.js
```

- **What I found:** A function called `preMoveCheck()`.
    

```javascript
function preMoveCheck(from, to, promotion) {
  const probe = new Chess(game.fen());
  let result;
  try {
    result = probe.move({ from, to, promotion: promotion || undefined });
  } catch (e) {
    result = null;
  }
  if (result && probe.isCheckmate()) {
    showSystemNotice("I'll shut down your PC if you play that.");
    return false;
  }
  return true;
}
```

- **Why it matters:** This was the exact reason my winning move was being blocked.
    

The application creates a temporary chess board, simulates my move, and checks:

```javascript
probe.isCheckmate()
```

If the move results in checkmate, it returns `false` and refuses to let the move continue.

So the restriction wasn't coming from the server. **The browser itself was stopping me.**

---

### Step 3 — Finding How Moves Reach the Server

- **What I did:** Instead of focusing only on the blocked function, I searched further through `app.js` to see how normal moves were actually sent to the backend.
    
- **Command/Tool used:** JavaScript source inspection
    

I found:

```javascript
async function sendMove(from, to, promotion) {
  locked = true;
  let data;
  try {
    const res = await fetch('/api/move', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        from,
        to,
        promotion: promotion || undefined
      })
    });
    data = await res.json();
  } catch (e) { ... }

  finalize(data);
}
```

- **What I found:** The application sends moves to:
    

```text
/api/move
```

using a simple JSON POST request.

- **Why it matters:** This gave me another way into the game logic.
    

The browser was checking whether the move was allowed **before** calling `/api/move`.

That meant I didn't necessarily have to use the browser's move interface at all.

---

### Step 4 — Understanding the JavaScript Scope

There was one small problem.

I couldn't simply open the browser console and run:

```javascript
sendMove('a1', 'a8')
```

The script was loaded using:

```html
type="module"
```

so its functions weren't available as normal global functions from the console.

But honestly, that didn't matter.

I already knew what the backend endpoint expected.

---

### Step 5 — Sending the Move Directly

- **What I did:** Recreated the HTTP request manually with `curl`, completely bypassing the browser's `preMoveCheck()` function.
    
- **Command/Tool used:**
    

```bash
curl -X POST http://<MACHINE_IP>/api/move \
     -H "Content-Type: application/json" \
     -d '{"from":"a1","to":"a8"}'
```

- **What I found:** The server accepted the move and returned the victory response.
    

```json
{
  "ok": true,
  "move": "a1a8",
  "fen": "R5k1/5ppp/8/8/8/8/5PPP/6K1 b - - 1 1",
  "status": "checkmate",
  "turn": "b",
  "winner": "white",
  "flag": "THM{cl13nt_s1d3_ch3ckm4t3}"
}
```

- **Why it matters:** I completely skipped the client-side restriction.
    

The browser said:

> "Nope, you're not allowed to play that."

The backend said:

> "Sure, that's a valid move."

---

## 🛠 Commands Used

|Command / Tool|What It Does|
|---|---|
|`curl -s http://<MACHINE_IP>/js/app.js`|Downloads the application's JavaScript source|
|Browser Developer Tools|Inspects the application's HTML and JavaScript|
|`preMoveCheck()`|Client-side function that blocks checkmate moves|
|`/api/move`|Backend API endpoint used to submit chess moves|
|`curl -X POST ...`|Sends the winning move directly to the backend|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|Challenge flag|`THM{cl13nt_s1d3_ch3ckm4t3}`|

---

## 📎 Resources Used

- TryHackMe challenge
    
- Browser Developer Tools
    
- JavaScript source code provided by the target application
    

---