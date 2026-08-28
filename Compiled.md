# Compiled

**Platform:** TryHackMe  
**Difficulty:** Easy  
**Category:** Reverse Engineering / Binary Analysis  
**Status:** ✅ Completed

---

## 📌 What Was This Challenge About?

This challenge was a small introduction to **reverse engineering**.

The goal was pretty simple: we get a compiled binary with no source code, figure out what it's doing, and then find the correct password.

At first, all we know is:

```text
Password:
```

So naturally, I tried to figure out what the program was actually checking instead of just guessing passwords forever. 😎

This challenge mainly taught me:

- How to inspect a compiled binary
    
- How `scanf()` format strings work
    
- How `strcmp()` is used to compare strings
    
- How static analysis can reveal exactly what a program is checking
    

---

# 🔍 Step 1 — Checking the Binary

First, I wanted to know what I was actually dealing with.

```bash
file compiled
```

This identified the file as a compiled executable.

I then ran it:

```bash
./compiled
```

And got:

```text
Password:
```

I obviously tried a random password first:

```text
test
```

And the program responded:

```text
Try again!
```

So there was definitely some sort of password check happening.

At this point, guessing wasn't really going to get me anywhere, so I decided to inspect the binary.

---

# 🧭 Step 2 — Static Analysis

I opened the binary in **Ghidra** and looked through the decompiled code.

The important part looked roughly like this:

```c
undefined8 main(void)
{
    int iVar1;
    char local_28[32];

    fwrite("Password: ", 1, 10, stdout);
    __isoc99_scanf("DoYouEven%sCTF", local_28);

    iVar1 = strcmp(local_28, "__dso_handle");
    if ((-1 < iVar1) && (iVar1 = strcmp(local_28, "__dso_handle"), iVar1 < 1)) {
        printf("Try again!");
        return 0;
    }
    
    iVar1 = strcmp(local_28, "_init");
    if (iVar1 == 0) {
        printf("Correct!");
    } else {
        printf("Try again!");
    }

    return 0;
}
```

And this was where the challenge basically gave me the answer. 😂

---

# 🧠 Step 3 — Understanding the `scanf()` Format

The line I cared about most was:

```c
scanf("DoYouEven%sCTF", local_28);
```

At first this looks a little weird.

The important part is:

```text
DoYouEven%sCTF
```

The `%s` is the format specifier that tells `scanf()` to read a string.

So the program expects the input to have this structure:

```text
DoYouEven + [something] + CTF
```

The `[something]` gets stored in:

```c
local_28
```

So if I entered:

```text
DoYouEvenHELLOCTF
```

then the string captured by `%s` would be:

```text
HELLO
```

That's the important part.

---

# 🔎 Step 4 — Following the Comparisons

Next, I looked at what the program did with `local_28`.

First it compares the input against:

```text
__dso_handle
```

If it matches, the program says:

```text
Try again!
```

So that's a dead end.

Then we get the important comparison:

```c
strcmp(local_28, "_init")
```

If the result is `0`, the program prints:

```text
Correct!
```

So now we know exactly what the program wants:

```text
local_28 = _init
```

Since `_init` is the part captured by `%s`, I just needed to put it between the two fixed pieces of the format string.

That gives:

```text
DoYouEven_init
```

---

# 🏆 Step 5 — Testing the Password

I ran the program again:

```bash
./compiled
```

Then entered:

```text
DoYouEven_init
```

And got:

```text
Correct!
```

**BOOM.** 🎉

The password was correct.

---

# 🛠️ Tools & Commands Used

|Tool / Command|What It Does|
|---|---|
|`file compiled`|Identifies the type and basic information about the binary|
|`./compiled`|Executes the binary|
|Ghidra|Used to decompile and analyze the binary|
|`strcmp()`|Compares two strings in C|
|`scanf()`|Reads formatted input from the user|

---

# 🚩 Solution

| Item             | Value            |
| ---------------- | ---------------- |
| `%s` value       | `_init`          |
| Correct password | `DoYouEven_init` |

---

# 💡 What I Learned

- **Static analysis is REALLY useful.** Instead of trying thousands of passwords, I could look at the program's logic and see exactly what it was checking.
    
- I got a better understanding of how **`scanf()` format strings** work, especially how `%s` captures part of an input.
    
- I learned that **`strcmp()` returning `0` means the strings are equal**, which is important when reading decompiled C code.
    
- Even tiny binaries can teach useful reverse-engineering concepts. Once you understand what the code is doing, the "mystery" password usually isn't much of a mystery anymore. 😎
    

---

# ❓ What Confused Me / What I Want to Research Next

The part I want to understand better is how the decompiler translates the original C program into pseudo-C.

Some of the Ghidra output looked pretty strange compared to normal C, especially the `undefined8`, `__isoc99_scanf`, and the way the `strcmp()` checks were represented.

I'd like to learn more about:

- How C gets compiled into assembly
    
- How Ghidra reconstructs C-like code
    
- x86/x64 assembly basics
    
- How function arguments and return values work at the assembly level
    
- How to recognize common C functions when looking directly at assembly
    

---

# 🔗 Linked Notes

- [[Reverse Engineering]]
    
- [[Ghidra]]
    
- [[C Programming]]
    
- [[Static Analysis]]
    
- [[Binary Analysis]]
    
- [[strcmp]]
    
- [[scanf]]
    

---

# 🏁 Final Thoughts

This was a really nice little introduction to reverse engineering.

The program basically looked like:

> **"Password?"**

And instead of guessing, I went:

> **"Nah, let me read your brain first."** 😂

Once I found the `scanf()` format and followed the `strcmp()` checks, the password was basically sitting right there.

```text
DoYouEven_initCTF
```

**Challenge completed. 🚀**

_Report written in [[TryHackMe Vault]] — part of my ethical hacking journey 🛡️_