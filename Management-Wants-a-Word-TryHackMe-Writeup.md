# 🏨 TryHackMe — Management Wants a Word

> **Hacker Holidays · The Byte Lotus Hotel**

A Windows forensics challenge focused on **artifact correlation, Chrome credential recovery, Windows DPAPI, encrypted containers, and document forensics**.

## 📌 Challenge Information

| Category   | Details                 |
| ---------- | ----------------------- |
| Platform   | TryHackMe               |
| Category   | Forensics               |
| Difficulty | Hard                    |
| Points     | 120                     |
| Room       | Management Wants a Word |
| Target     | Vera's forensic triage  |

---

## 🛎️ Scenario

A guest named **Vera** left her laptop behind after checking out of Room 214.

The hotel IT team performed a full forensic triage before wiping the machine.

The objective was to investigate the artifacts left behind, recover a password, follow the evidence, and discover what Vera was hiding.

---

## 🔎 Investigation Methodology

The challenge required correlating multiple artifacts rather than finding a single obvious file.

The investigation followed this chain:

```text
Chrome History
      ↓
SecureVault Portal
      ↓
Chrome Saved Credentials
      ↓
SAM + SYSTEM
      ↓
NTLM Hash
      ↓
Password Recovery
      ↓
Windows DPAPI
      ↓
Chrome Credential Decryption
      ↓
Encrypted "backup"
      ↓
VeraCrypt Container
      ↓
FAT32 Filesystem
      ↓
Financial Documents
      ↓
CSV Clue
      ↓
Invoice PDF
      ↓
Embedded Image
      ↓
Flag
```

---

## 1. 🌐 Chrome History

Chrome history revealed a self-hosted application:

```text
http://bytelotus.thm:8080/login
```

The application was identified as the **SecureVault Portal**.

This gave us the first major lead.

---

## 2. 🔐 Chrome Saved Credentials

Chrome's `Login Data` database contained credentials for the SecureVault application.

```text
Username:
VeraSecretVault
```

The password was encrypted and stored as a `v10` credential.

Therefore, simply extracting the SQLite database wasn't enough.

---

## 3. 🪟 Windows SAM & SYSTEM

The forensic image contained the Windows registry hives:

```text
SAM
SYSTEM
SECURITY
```

The SAM and SYSTEM hives were used to recover Vera's local account NTLM hash.

Example:

```bash
secretsdump.py \
  -sam SAM \
  -system SYSTEM \
  -security SECURITY \
  LOCAL
```

The recovered NTLM hash was then cracked using Hashcat:

```bash
hashcat -m 1000 hash.txt /usr/share/wordlists/rockyou.txt
```

Recovered Windows password:

```text
minivera
```

---

## 4. 🔑 Windows DPAPI

The recovered Windows password allowed the DPAPI-protected material belonging to Vera's account to be processed.

The relevant chain was:

```text
Windows Password
       ↓
DPAPI MasterKey
       ↓
Chrome Encryption Key
       ↓
Chrome Login Data
       ↓
Saved SecureVault Credential
```

The SecureVault credentials were recovered as:

```text
Username:
VeraSecretVault

Password:
Wh4t1sV3raD0inG0nTh1sH0st
```

---

## 5. 📦 Suspicious `backup` File

A suspicious approximately **100 MB** file was discovered in Vera's Documents directory:

```text
C:\Users\vera\Documents\backup
```

The file had no conventional extension.

Further analysis showed that it was a **VeraCrypt container**.

The recovered SecureVault password was used as the container passphrase.

The container header's CRC32 validation provided confirmation that the password/key derivation was correct.

---

## 6. 💾 Hidden Filesystem

After decrypting the container, a FAT32 filesystem was discovered.

Inside was:

```text
secret_financial_documents/
```

The directory contained files including:

```text
invoice.pdf
transactions_q3.csv
```

---

## 7. 📊 CSV Investigation

One unusual CSV entry stood out:

```text
Internal Adjustment
Image asset correction
$0.00
```

The reference to an **image asset** suggested that the invoice PDF itself needed deeper inspection.

---

## 8. 📄 Embedded Image in PDF

The PDF was examined for embedded images.

```bash
pdfimages -list invoice.pdf
```

The images were then extracted:

```bash
pdfimages -png invoice.pdf extracted
```

The extracted PNG contained the final clue and revealed the challenge flag.

---

## 🧰 Tools Used

```text
SQLite
Chrome Forensic Artifacts
Impacket
secretsdump
Hashcat
Windows DPAPI
VeraCrypt
Python
pdfimages
Linux CLI
```

---

## 🧠 Key Takeaways

### Artifact correlation

The most important lesson was that no single artifact contained the complete answer.

Each artifact pointed toward the next:

```text
History → Portal
Portal → Credentials
SAM/SYSTEM → Windows Password
Windows Password → DPAPI
DPAPI → Chrome Password
Chrome Password → VeraCrypt
VeraCrypt → Documents
CSV → PDF Image
PDF Image → Flag
```

### Forensic mindset

When investigating a forensic image, always ask:

> **"What does this artifact tell me, and where can it lead next?"**

Unusual files, browser artifacts, registry hives, metadata, and seemingly insignificant records can all become important when correlated.

---

## 🏁 Flag

<details>
<summary>Click to reveal the flag</summary>

```text
THM{1t_w4s_V3r4_A11_Al0ng?!}
```

</details>

---

## 📚 Skills Practiced

* Windows Forensics
* Browser Artifact Analysis
* Chrome Credential Investigation
* Windows Registry Analysis
* NTLM Hash Extraction
* Password Cracking
* Windows DPAPI
* Encrypted Container Analysis
* FAT32 Filesystem Analysis
* PDF Forensics
* Embedded Object Extraction
* Evidence Correlation

---

**Author:** Shivam
**Platform:** TryHackMe
**Category:** Digital Forensics / DFIR
