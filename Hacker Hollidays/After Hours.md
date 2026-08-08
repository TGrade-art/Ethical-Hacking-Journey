
---

# After Hours

**Platform:** TryHackMe 
**Date:** 2026-07-08 
**Difficulty:** Medium 
**Category:** Windows Forensics / Incident Response 
**Room URL:** https://tryhackme.com/room/hh-afterhours-b090d1f0
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> This room was about Windows WMI repository forensics and manually deobfuscating .NET malware. The scenario was an incident response situation where an attacker planted a fileless persistence mechanism inside the WMI database and I had to dig it out, decode it, and reverse engineer it to find the flag.

---

## 🔍 Reconnaissance

No live host to scan here, this was an offline forensics challenge, so I went straight to analyzing the provided disk artifacts.

```bash
ls -la after-hours-forensics-hh/challenge_attachments
```

| Port | Service           | Version                | Notes                                      |
| ---- | ----------------- | ---------------------- | ------------------------------------------ |
| N/A  | Offline Artifacts | Windows WMI Repository | Analyzed static OBJECTS.DATA file directly |
|      |                   |                        |                                            |

---

## 🧭 Steps I Took

### Step 1 — Extracting the Forensic Artifacts

- **What I did:** I accessed the password-protected forensic package to get at the raw system files.
- **Command/Tool used:**

```bash
# 7z x after-hours.7z
# Passphrase: Aft3rH0ursAtt4chm3ntP4ss
```

- **What I found:** A set of Windows WMI database files — `INDEX.BTR`, `MAPPING1.MAP`, and the juicy one: `OBJECTS.DATA`.
- **Why it matters:** `OBJECTS.DATA` is where Windows stores CIM class definitions and instances. It's also where attackers hide fileless malware because most antivirus tools don't look there.

### Step 2 — Carving Out the Hidden Payload

- **What I did:** Since I didn't have the custom Python parser tool the room expected, I improvised and used `strings` combined with `awk` to pull out anything suspiciously long from the binary.
- **Command/Tool used:**

```bash
# strings OBJECTS.DATA | awk 'length($0) > 1000' | head -n 1 > pure_payload.txt
```

- **What I found:** A massive Base64 string and an encoded PowerShell command targeting a fake WMI class called `ROOT\cimv2:Win32_HardwareTelemetry`.
- **Why it matters:** This is how fileless malware works, no executable on disk, just a blob of encoded nastiness sitting inside a database field, waiting to be loaded straight into RAM.

### Step 3 — Decoding and Decompressing the Malware

- **What I did:** I stripped the Base64 layer and decompressed the payload. The tricky part was that the attacker used raw DEFLATE compression without the standard zlib headers to trip up basic extraction scripts.
- **Command/Tool used:**

```bash
# python3 -c "
import base64, zlib
b64_str = open('pure_payload.txt').read().strip()
compressed = base64.b64decode(b64_str)
open('malware_payload.dll', 'wb').write(zlib.decompress(compressed, -15))
"
```

- **What I found:** A recovered `.NET` DLL — `malware_payload.dll`.
- **Why it matters:** The `-15` window bits flag is what handles raw DEFLATE without headers. Without that you just get errors. It took me a minute to figure that one out.

### Step 4 — Reverse Engineering the DLL in ILSpy

- **What I did:** I opened the recovered DLL in ILSpy to decompile it back to readable C#. One thing that tripped me, ILSpy was showing empty code paths until I switched the decompiler target from C# 8.0 down to C# 4.0. After that everything appeared.
- **Command/Tool used:**

```bash
# ./ILSpy
```

- **What I found:** The decompiled C# logic checks the target machine's hostname against the string `bytelotusdc`. If it matches, it fires this command:

```
/c net user patch VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9 /add
```

That Base64 string at the end looked very interesting 🧐

### Step 5 — Decoding the Flag

- **What I did:** That Base64 string inside the `net user` command was clearly hiding something. One decode later and BOYAH 🎉
- **Command/Tool used:**

```bash
# echo "VEhNe1A0dGNoX29wM25lZF90aDNfQmFjS2QwMHJ9" | base64 -d
```

- **What I found:** The flag hiding inside the malware's own backdoor command!

---

## 🛠 Commands Used

|Command|What It Does|
|---|---|
|`unzip after-hours.7z`|Extracts the password-protected forensic artifact package|
|`strings OBJECTS.DATA`|Pulls readable text out of the binary WMI database file|
|`awk 'length($0) > 1000'`|Filters for suspiciously long strings — Base64 payloads are huge|
|`base64 -d`|Decodes Base64 encoded data|
|`zlib.decompress(data, -15)`|Decompresses raw DEFLATE data without standard zlib headers|
|`./ILSpy`|Opens the .NET decompiler to reverse engineer the DLL back to C#|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|Final Flag|`THM{P4tch_op3ned_th3_BacKd00r}`|

---

## 📎 Resources Used

- TryHackMe Hacker Holidays 2026 Walkthrough Track Documentation
- Microsoft Learn: WMI Repository Architecture
- CyberChef Deflate/Inflate Operations

---