
---

# Management Wants a Word — Forensics (Hacker Holidays Day 14)

**Platform:** TryHackMe 
**Date:** 2026-08-10 
**Difficulty:** Hard 
**Category:** Windows Forensics / DPAPI / Cryptography 
**Room URL:** https://tryhackme.com/room/managementwantsaword 
**Status:** ✅ Completed

---

## 📌 What Was This Room About?

> This room was about forensic analysis of Windows KAPE triage artifacts, recovering DPAPI keys, extracting Chrome credentials, and decrypting a custom VeraCrypt volume. The goal was to pivot through everything Vera left behind and crack open a hidden PDF invoice containing the flag.

---

## 🔍 Reconnaissance

No live host to scan here, this was an offline forensics challenge using KAPE triage artifacts from `/KAPE/C/`. So instead of nmap I went straight to poking around the user profile directories and registry structures.

|Artifact / Path|Type|Notes|
|---|---|---|
|`AppData/Roaming/Microsoft/Protect/S-1-5-21-.../c90719ef-...`|DPAPI Master Key|User DPAPI master key file|
|`AppData/Local/Google/Chrome For Testing/.../Login Data`|SQLite Database|DPAPI-encrypted Chrome credential store|
|`C/Users/vera/Documents/backup`|VeraCrypt Container|Encrypted 100 MiB hidden storage volume|

---

## 🧭 Steps I Took

### Step 1 — Generating DPAPI Prekeys

- **What I did:** Used `pypykatz` to compute the DPAPI prekey using Vera's SID and user password (`minivera`). One thing that got me confused was that to run it, I had to be under Python 3.12 because `pypykatz` throws PEP 604 type union errors under Python 3.8. 😤
- **Command/Tool used:**

```bash
# /usr/bin/python3 -m pypykatz dpapi prekey password "S-1-5-21-2529683458-431225740-1723070931-1000" 'minivera' -o prekeys.txt
```

- **What I found:** Successfully derived SHA1 and PBKDF2 prekey hashes saved to `prekeys.txt`.
- **Why it matters:** Windows DPAPI requires prekeys derived from the user password and SID before you can even think about touching the master keys. No prekeys, no progress.

### Step 2 — Decrypting the DPAPI Master Key

- **What I did:** Located Vera's DPAPI master key GUID (`c90719ef-5b98-474e-b934-136d606a702a`) inside the `Protect/<SID>/` directory and fed it to `pypykatz` along with the prekeys file.
- **Command/Tool used:**

```bash
# /usr/bin/python3 -m pypykatz dpapi masterkey \
  "C/Users/vera/AppData/Roaming/Microsoft/Protect/S-1-5-21-2529683458-431225740-1723070931-1000/c90719ef-5b98-474e-b934-136d606a702a" \
  prekeys.txt -o masterkeys.json
```

- **What I found:** Decrypted master key material saved to `masterkeys.json`. 🎉
- **Why it matters:** Master keys are what unlock application-specific DPAPI blobs — like saved browser passwords. Without this step you're going nowhere.

### Step 3 — Raiding Chrome's Saved Passwords

- **What I did:** Passed `masterkeys.json` into `pypykatz` along with Chrome's `Local State` and `Login Data` files to extract whatever credentials Vera had saved.
- **Command/Tool used:**

```bash
# /usr/bin/python3 -m pypykatz dpapi chrome masterkeys.json \
  "C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Local State" \
  --logindata "C/Users/vera/AppData/Local/Google/Chrome For Testing/User Data/Default/Login Data"
```

- **What I found:** Vera's saved password = `Wh4t1sV3raD0inG0nTh1sH0st`
- **Why it matters:** This password turned out to be the passphrase protecting the VeraCrypt container sitting in her Documents folder.

### Step 4 — Cracking Open the VeraCrypt Container

- **What I did:** The `backup` file in Vera's Documents was a VeraCrypt container. I wrote a custom Python script to handle sector-by-sector AES-XTS decryption because standard tools weren't cooperating. The tricky part: AES-XTS requires updating the tweak value for every individual 512-byte sector. I initially tried reading in 512 KB chunks without updating the tweak counter and got a completely corrupted image.
- **Command/Tool used:**

```python
# vcdec.py — sector-by-sector XTS decryption
import sys
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.ciphers import Cipher, algorithms, modes
from cryptography.hazmat.primitives.kdf.pbkdf2 import PBKDF2HMAC

backup_file = "backup"
password = b"Wh4t1sV3raD0inG0nTh1sH0st"
output_img = "vault.img"

with open(backup_file, "rb") as f:
    head = f.read(512)
    salt, enc = head[:64], head[64:512]

hk = PBKDF2HMAC(hashes.SHA512(), 64, salt, 500_000).derive(password)

c_head = Cipher(algorithms.AES(hk), modes.XTS((0).to_bytes(16, "little"))).decryptor()
dec_head = c_head.update(enc) + c_head.finalize()
assert dec_head[:4] == b"VERA", "Header decryption failed!"

keys = dec_head[192:192+64]
start = int.from_bytes(dec_head[44:52], "big")

with open(backup_file, "rb") as f, open(output_img, "wb") as out:
    f.seek(start)
    unit = start // 512
    while sector := f.read(512):
        if len(sector) < 512:
            break
        c = Cipher(algorithms.AES(keys), modes.XTS(unit.to_bytes(16, "little"))).decryptor()
        out.write(c.update(sector) + c.finalize())
        unit += 1
```

```bash
# /usr/bin/python3 vcdec.py
```

- **What I found:** A valid FAT32 disk image = `vault.img` 🎉
- **Why it matters:** Each 512-byte sector needs its own tweak value. Read in bigger chunks without updating the counter and you corrupt everything. Learned that the hard way.

### Step 5 — Extracting the PDF and Getting the Flag

- **What I did:** I used `icat` to pull inode 40 out of the FAT32 image, which turned out to be `important_invoice_byte_lotus.pdf`. But the flag wasn't readable as plain text, it was embedded as raw pixels inside a compressed PDF image stream. So I wrote a quick Python script to decompress the FlateDecode stream and render it as a PNG.
- **Command/Tool used:**

```bash
# icat vault.img 40 > important_invoice_byte_lotus.pdf
```

```python
# /usr/bin/python3 -c '
import re, zlib
from PIL import Image

data = open("important_invoice_byte_lotus.pdf", "rb").read()
match = re.search(rb"/Subtype\s*/Image.*?stream\r?\n(.*?)endstream", data, re.S)
if match:
    raw = zlib.decompress(match.group(1))
    Image.frombytes("RGB", (636, 724), raw).save("invoice.png")
'
```

- **What I found:** `invoice.png` rendered the full invoice with the flag right there on Line 1. And that was that 😈
- **Why it matters:** The flag was hidden as raw pixel data inside a compressed PDF stream specifically to dodge string-matching forensic tools. 

---

## 🛠 Commands Used

|Command|What It Does|
|---|---|
|`pypykatz dpapi prekey password ...`|Generates DPAPI prekeys from the user password and SID|
|`pypykatz dpapi masterkey ...`|Decrypts the DPAPI master key using derived prekeys|
|`pypykatz dpapi chrome ...`|Extracts saved Chrome passwords from the Login Data database|
|`/usr/bin/python3 vcdec.py`|Runs sector-by-sector AES-XTS decryption on the VeraCrypt container|
|`icat vault.img 40 > important_invoice_byte_lotus.pdf`|Extracts inode 40 from the FAT32 disk image|
|`zlib.decompress(match.group(1))`|Decompresses the FlateDecode PDF image stream into raw RGB pixels|

---

## 🚩 Flags Found

|Flag|Value|
|---|---|
|Room Flag|`THM{1t_w4s_V3r4_A11_Al0ng?!}`|

---

## 📎 Resources Used

- [pypykatz Repository](https://github.com/skelsec/pypykatz)
- VeraCrypt Format Specification
- Sleuthkit Tool Documentation (fls / icat)

---