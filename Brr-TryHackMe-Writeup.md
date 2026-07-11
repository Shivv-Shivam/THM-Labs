# 🥶 TryHackMe — Brr (Writeup)

**Room:** https://tryhackme.com/room/brr (Premium)
**Difficulty:** 🟢 Easy
**Category:** OT / ICS
**Tags:** `ot` `ics` `scada` `modbus` `scadabr` `novnc`

## 📖 Room Story

We need to assess an OT (Operational Technology) environment. There's a SCADA panel that will lead us straight to the cold plant/process where the flag is hidden. Just follow the panel, chase the "chill" all the way down. ❄️

---

## 🔍 Step 1: Enumeration (Nmap Scan)

First, run a full TCP port scan to see what services are running on the machine:

```bash
nmap -Pn -n -p- --min-rate 1500 -T4 --open <MACHINE_IP>
```

This scan shows the following open ports:

| Port | Service | What it is |
|------|---------|------------|
| 22 🔑  | SSH     | Normal remote login |
| 80 🌐  | HTTP    | noVNC web front-end (WebSockify) |
| 5020 ⚙️ | Modbus? | PLC's Modbus TCP endpoint |
| 5901 🖥️ | VNC     | Graphical HMI |
| 8080 🏭 | HTTP-proxy | ScadaBR (SCADA web panel, running on Tomcat) |

Now open both ports in your browser to confirm:

- Open `http://<MACHINE_IP>/` in your browser → there is error no any web page on port 80.
- Open `http://<MACHINE_IP>:8080/` in your browser → this redirects to the ScadaBR login page, and the tab title shows "ScadaBR CTF" — confirming it's the ScadaBR panel.

> 💡 **Important cue:** Port `5020` is a classic alternate port for Modbus TCP in OT/ICS challenges. This is a strong signal that the "chill" (flag) is hiding here.

---

## 🔓 Step 2: ScadaBR Panel Access (Default Credentials)

[ScadaBR](https://www.scadabr.com.br/) is a web-based SCADA/HMI tool built on the Mango platform. It's well known for a common weakness — default credentials are often left unchanged.

Same case here — we get in easily using ScadaBR's default admin login. (Default creds are easy to find via Google or the room material, and they're also listed in ScadaBR's own documentation.)

Log in through the browser:

1. Open `http://<MACHINE_IP>:8080/ScadaBR/login.htm`.
2. Enter `admin` in the username field, and ScadaBR's default password `admin` in the password field.
3. Click login. ✅

Once logged in, you'll be redirected straight to the **Watch List** page — meaning you're now inside the admin panel. 🎉

---

## 📡 Step 3: Find the Data Source

Inside the admin panel, go to the **Data Sources** section. You'll find one data source configured:

```
Data source: Modbus IP
Host: plc:5020
```

So ScadaBR isn't holding the flag itself — it's just a panel telling us where the real process data (the PLC) lives. Now we need to talk directly to the PLC over Modbus TCP on port 5020. 🏭➡️⚙️

---

## 🧊 Step 4: Reading Data From the PLC (Modbus TCP)

In most OT/ICS CTF challenges, the flag is stored directly in a device's registers. `pymodbus` isn't available on the machine, but the Modbus TCP protocol is simple enough to build by hand — we just need a header and a Read Holding Registers request (function code `0x03`).

Python script:

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

Each 16-bit holding register stores one ASCII character (high byte `0x00`, low byte the actual character). As soon as you run the script, the registers get decoded and the flag gets printed — something like:

```
Registers: 0x0054 0x0048 0x004d 0x007b ...   →  T H M {
Decoded  : THM{...}
```

🚩 Flag found — it was hiding inside the cold PLC's holding registers. 🥶

---

## ✅ Summary (TL;DR)

1. 🔍 **Recon**: Nmap shows a ScadaBR panel (8080) and a Modbus endpoint (5020).
2. 🔓 **Access**: Logged in as admin using ScadaBR's default credentials.
3. 📡 **Pivot**: The Data Sources config revealed the PLC is on `5020`.
4. 🧊 **Exploit**: Read the PLC's holding registers directly over Modbus TCP — flag was right there.

---

## 🛡️ Defensive Takeaways

- 🔑 **Never leave default credentials unchanged** — it's an open door. Change default creds on HMI/SCADA panels right away.
- ⚠️ **Modbus has no authentication or encryption** — anyone who can reach the port can read or write registers. Never expose Modbus to a general network or the internet.
- 🧱 **Segment OT and IT networks** — there should be a firewall/DMZ between them.
- 🚫 **Don't store sensitive data in PLC registers** — any Modbus client can read them.

---

*Solve the room yourself before reading this writeup — it contains spoilers. Happy hacking!* 🕵️‍♂️❄️
