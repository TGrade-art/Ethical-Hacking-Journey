# The Game

**Platform:** TryHackMe  
**Difficulty:** Easy  
**Category:** Reverse Engineering / CTF  
**Room URL:** https://tryhackme.com/room/hfb1thegame
**Status:** ✅ Completed

---

## 📌 What Was This Challenge About?

This challenge was a pretty simple introduction to **static analysis**.

The goal was basically to take a Windows executable, look through the readable strings inside it, and find the hidden TryHackMe flag.

No debugger.  
No complicated reverse engineering.  
No unpacking.

Just:

```bash
strings
```

Sometimes the easiest tool is the right tool. 😎

---

# 🔍 Getting the Task Files

First, I downloaded the task files from TryHackMe using the **Download Task Files** button.

The downloaded file was a ZIP archive with a name similar to:

```text
Tetrix.exe-1741979048280.zip
```

The exact filename can be different depending on the download.

I extracted it with:

```bash
unzip Tetrix.exe-1741979048280.zip
```

Then I checked what was inside:

```bash
ls -l
```

This showed the executable:

```text
Tetrix.exe
```

---

# 🧭 Steps I Took

## Step 1 — Checking the Binary

At this point I had the executable, so my first thought was:

> "Do I really need to reverse engineer this thing?" 😂

Before opening it in a debugger or using a more complicated tool, I decided to try the simplest option first.

That's where `strings` comes in.

---

# Step 2 — Using `strings`

I ran:

```bash
strings -n 6 Tetrix.exe
```

The `strings` command searches a binary for sequences of readable characters.

The `-n 6` option tells it to only display strings that are at least **6 characters long**.

This is useful because executables contain a LOT of random-looking data, and filtering out tiny strings makes the output much easier to read.

---

# Step 3 — Searching Specifically for the Flag

Instead of scrolling through hundreds or thousands of strings manually, I piped the output into `grep`:

```bash
strings -n 6 Tetrix.exe | grep -i "thm{"
```

This command is doing two things:

### `strings -n 6 Tetrix.exe`

Extracts readable strings from the executable.

### `|`

The pipe sends the output of `strings` directly into the next command.

### `grep -i "thm{"`

Searches the output for `thm{`.

The `-i` makes the search case-insensitive, so it can find variations such as:

```text
THM{
thm{
ThM{
```

I searched for `THM{` because that's the format TryHackMe flags normally use.

---

# 🚩 Flag

And there it was.

```text
THM{flag_text_here}
```

The flag was sitting directly inside the executable as a readable string.

**No debugger required.** 😎

---

# 🛠 Commands Used

|Command|What It Does|
|---|---|
|`unzip <file>.zip`|Extracts the downloaded task archive|
|`ls -l`|Lists the extracted files|
|`strings -n 6 Tetrix.exe`|Extracts printable strings of at least 6 characters|
|`grep -i "thm{"`|Searches the output for the TryHackMe flag format|

---

# 💡 What I Learned

- **Always try the simple stuff first.** Before reaching for Ghidra, IDA, or a debugger, checking a binary with `strings` can sometimes solve the entire challenge.
    
- `strings` is useful for finding **hard-coded information** inside compiled programs, such as URLs, usernames, passwords, error messages, and CTF flags.
    
- Using a pipe with `grep` makes command-line analysis MUCH faster because I don't have to manually search through massive amounts of output.
    
- Not every binary requires complicated reverse engineering. Sometimes the developer accidentally leaves the important information sitting right there. 😂
    

---

# ❓ What I Want to Research Next

This challenge was pretty straightforward, so the obvious next step for me is learning what to do **when `strings` doesn't work**.

I want to get better with:

- **Ghidra**
    
- **x86/x64 assembly**
    
- **PE executable structure**
    
- **Static vs dynamic analysis**
    
- **Finding obfuscated strings**
    
- **Debugging with GDB/LLDB**
    
- **Basic malware/reverse-engineering techniques**
    

Because `strings` is great when the flag is just sitting there...

But I'm guessing the harder challenges won't be that nice. 💀

---

# 🏁 Final Thoughts

This was a perfect example of:

> **Don't overcomplicate things.**

I could have spent ages setting up a debugger and digging through assembly, but one command was enough:

```bash
strings -n 6 Tetrix.exe | grep -i "thm{"
```

**Flag found. Challenge done.** 🚀

_Report written in [[TryHackMe Vault]] — part of my ethical hacking journey 🛡️_