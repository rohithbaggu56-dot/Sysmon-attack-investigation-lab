# 🔴 Sysmon Attack Investigation Lab
## Attacking a Windows VM & Investigating Every Trace

![Sysmon](https://img.shields.io/badge/Sysmon-Endpoint%20Detection-0078D6?style=for-the-badge&logo=windows&logoColor=white)
![Kali Linux](https://img.shields.io/badge/Kali_Linux-Attacker-557C94?style=for-the-badge&logo=kalilinux&logoColor=white)
![Metasploit](https://img.shields.io/badge/Metasploit-Exploitation-2596CD?style=for-the-badge&logo=metasploit&logoColor=white)
![MITRE ATT&CK](https://img.shields.io/badge/MITRE-ATT%26CK-red?style=for-the-badge)
![VirusTotal](https://img.shields.io/badge/VirusTotal-Hash%20Analysis-394EFF?style=for-the-badge)

A hands-on Sysmon investigation lab where I simulated a realistic attack chain against a Windows 10 VM from Kali Linux, then investigated every trace left behind using Sysmon event logs building a complete attack timeline from endpoint evidence alone.

This lab was inspired by TCM Security's tutorial:
"I Hacked Myself & Analyzed It with Sysmon"
https://www.youtube.com/watch?v=OAuVYbn1m3A

---

## 🎯 Objective

Understand why Sysmon provides deeper endpoint visibility than native 
Windows Event Viewer, by:

- Executing a realistic multi-stage attack from Kali Linux
- Capturing forensic evidence through Sysmon event logs
- Building a complete attack timeline from endpoint logs alone
- Performing hash-based IOC validation using VirusTotal

---

## ❓ Why Sysmon Over Windows Event Viewer?

Windows Event Viewer told me a process ran.

Sysmon told me:
- What ran
- What spawned it
- What it connected to
- What it downloaded
- What hash it had
- Which MITRE ATT&CK technique it matched

Same machine. Completely different depth. That difference is what 
makes or breaks a real SOC investigation.

---

## 🏗️ Lab Environment

| Component | Role |
|-----------|------|
| Kali Linux VM | Attacker machine |
| Windows 10 VM | Victim machine with Sysmon installed |
| Metasploit Framework | Payload generation and exploitation |
| Sysmon | Endpoint detection and logging |
| Windows Event Viewer | Log analysis interface |
| VirusTotal | Hash-based IOC validation |

**Network:** Both VMs on isolated internal LAN (192.168.1.0/24)
- Kali: 192.168.1.101
- Windows 10: 192.168.1.102

---

## ⚔️ Attack Chain

### Stage 1 – Initial Access (T1204 User Execution)

Generated a Meterpreter reverse HTTP payload disguised as 
`FreeGamePasses.exe` using Metasploit.

Set up listener on Kali:
- LHOST: 192.168.1.101
- LPORT: 31337

Once the victim executed the file, a live Meterpreter session 
opened on Kali with full remote access.

**Screenshot:** Kali – Meterpreter session established

<img width="1911" height="1080" alt="Screenshot 2026-04-04 133346" src="https://github.com/user-attachments/assets/0f3a9d19-3c6e-45b0-9785-b92f7e441326" />

### Stage 2 – Discovery (T1018 Remote System Discovery)

Ran basic enumeration commands through the Meterpreter shell:
- `whoami` – confirmed running as victim user
- `hostname` – identified target machine
- `systeminfo` – gathered full system information

**Screenshot:** Kali – discovery commands and backdoor account creation

<img width="1911" height="1080" alt="Screenshot 2026-04-04 140616" src="https://github.com/user-attachments/assets/6c590f52-9a4c-485f-b342-99437752efe1" />

### Stage 3, 4 & 5 – Persistence, LOLBin Abuse & Credential Access
**(T1053 + T1136 + T1218 + T1202 + T1003)**

**Persistence – Scheduled Task & Backdoor Account**
- Created a scheduled task called `backdoortask` to re-execute the payload at every logon under SYSTEM context ensuring the attacker maintains access even after reboots.
- Also created a hidden backdoor user account and added it to the Administrators group for continued manual access if needed.

**LOLBin Abuse – certutil.exe**
- Used Windows' own built-in `certutil.exe` to download Mimikatz from GitHub and renamed to `notmimikatz.exe` to evade basic filename detection.
- Why this matters: certutil.exe is a legitimate trusted Windows binary for certificate management. Using it to download malware is called a LOLBin (Living Off the Land Binary)technique, it avoids triggering alerts because the process itself is trusted by the OS.

**Credential Access – Mimikatz**
- Executed Mimikatz with `sekurlsa::logonpasswords` to attempt credential extraction from memory. Even though it was renamed to notmimikatz.exe, Sysmon later caught it through internal file metadata proving renaming alone is not enough to evade detection.

**Screenshot:** Kali – scheduled task creation, certutil LOLBin download and Mimikatz execution all visible in one terminal

<img width="1911" height="1080" alt="Screenshot 2026-04-04 133427" src="https://github.com/user-attachments/assets/ebe05663-9aa3-4306-8bc9-098579604936" />

---
## 🔎 Sysmon Investigation – Event by Event

### Event ID 15 – File Stream Created
**MITRE:** T1189 Drive-by Compromise

FreeGamePasses.exe landed in the Downloads folder.Sysmon captured the Zone Identifier showing the file originated from Discord CDN, revealing exactly where 
the payload came from.

Hash captured:
SHA256: 8431e494b312cb548d774f606c43a4b94f186688a06f620e9b8849ed2c96d69b

**Screenshot:** Event ID 15 – file stream with origin URL visible
<img width="1903" height="1075" alt="Screenshot 2026-04-04 132551" src="https://github.com/user-attachments/assets/98ec7c7f-6a43-4d47-8b45-770bab800119" />

---

### Event ID 3 – Network Connection Detected
**MITRE:** T1571 Non-Standard Port

The moment FreeGamePasses.exe executed, Sysmon caught the outbound callback:

- Source: 192.168.1.102 (victim)
- Destination: 192.168.1.101:31337 (Kali listener)
- Process: FreeGamePasses.exe

An unknown executable making an outbound connection to a non-standard port is an immediate red flag in any SOC investigation.

**Screenshot:** Event ID 3 – network connection to Kali
<img width="1903" height="1075" alt="Screenshot 2026-04-04 134137" src="https://github.com/user-attachments/assets/b0ed3ae9-4f6c-423f-b08e-8f2f84706be1" />

---

### Event ID 1 – Process Create (Backdoor Account)
**MITRE:** T1136 Create Account

Sysmon captured the exact command line used to create the backdoor account — not just that net.exe ran, but the full arguments including the username and password.

Also captured the escalation command adding backdoor to the Administrators group.

**Screenshots:** Event ID 1 – net user and net localgroup commands
<img width="1903" height="1075" alt="Screenshot 2026-04-04 135140" src="https://github.com/user-attachments/assets/9cee0d1a-b114-4e30-bcca-ff1ccb9c486a" />

<img width="1903" height="1075" alt="Screenshot 2026-04-04 135304" src="https://github.com/user-attachments/assets/0357db4b-a705-4867-aab8-72f141cd2d12" />

---

### Event ID 22 – DNS Query (LOLBin Exposed)
**MITRE:** T1105 Ingress Tool Transfer

certutil.exe made a DNS query to github.com.

This single event exposed the LOLBin technique, a certificate utility has no legitimate reason to query GitHub. The process name, query destination, and timing all became pivot points for the investigation.

**Screenshot:** Event ID 22 – certutil DNS query to github.com
<img width="1903" height="1075" alt="Screenshot 2026-04-04 135344" src="https://github.com/user-attachments/assets/9804ac03-03a6-4e4a-bd69-a03cce404235" />

---

### Event ID 1 – Process Create (certutil downloading Mimikatz)
**MITRE:** T1202 Indirect Command Execution

Full command line captured showing certutil reaching out to the exact GitHub URL including the destination filename `notmimikatz.exe`.

**Screenshot:** Event ID 1 – certutil full command line visible
<img width="1903" height="1075" alt="Screenshot 2026-04-04 135523" src="https://github.com/user-attachments/assets/b4478bcc-9459-4148-a14f-1e3a2405e825" />

---

### Event ID 1 – Masquerading Detection (T1036)

The most impressive Sysmon catch of the lab.

Mimikatz was renamed to `notmimikatz.exe` to avoid detection. Sysmon flagged it anyway because internal file metadata still read:

- Description: mimikatz for Windows
- Company: gentilkiwi (Benjamin DELPY)
- OriginalFileName: mimikatz.exe

**You can rename a file. You cannot rename what it is.**

Sysmon tagged this as T1036 Masquerading automatically.

**Screenshot:** Event ID 1 – masquerading detection
<img width="1903" height="1075" alt="Screenshot 2026-04-04 135744" src="https://github.com/user-attachments/assets/97e60f5e-7950-4cb6-b2d9-ade4dfa9ca2d" />

---

## 🧪 Hash Validation — VirusTotal

Sysmon logged SHA256 hashes on every process execution.
Both hashes were submitted directly to VirusTotal.

| File | Detection | Verdict |
|------|-----------|---------|
| FreeGamePasses.exe | 42/71 vendors | Trojan/Meterpreter |
| mimikatz.exe | 62/71 vendors | Hacktool/Mimikatz |

**Screenshots:** VirusTotal results for both files
<img width="1920" height="1080" alt="Screenshot 2026-04-04 133554" src="https://github.com/user-attachments/assets/30c35323-2017-4f76-a8b6-caab4f51d026" />

<img width="1903" height="1075" alt="Screenshot 2026-04-04 135905" src="https://github.com/user-attachments/assets/1a5dd275-0e42-4ae3-8266-40554ba900e7" />

---

## 📋 Complete MITRE ATT&CK Mapping

| Technique ID | Name | How It Appeared |
|-------------|------|-----------------|
| T1204 | User Execution | Victim ran FreeGamePasses.exe |
| T1571 | Non-Standard Port | Meterpreter callback on port 31337 |
| T1189 | Drive-by Compromise | Payload delivered via Discord CDN |
| T1136 | Create Account | net user backdoor /add |
| T1053 | Scheduled Task | backdoortask created for persistence |
| T1218 | LOLBin Abuse | certutil.exe used to download Mimikatz |
| T1202 | Indirect Command Execution | certutil executing remote download |
| T1036 | Masquerading | notmimikatz.exe renamed to evade detection |
| T1003 | OS Credential Dumping | sekurlsa::logonpasswords via Mimikatz |

---

## 🧠 Key Takeaways

**1. Sysmon sees what Event Viewer misses**
Command lines, parent processes, hashes, network connections, DNS queries all invisible to basic Windows logging.

**2. LOLBins are dangerous because they blend in**
certutil.exe is trusted. Using it maliciously looks normal to basic monitoring. Behavioral detection through Sysmon exposed it.

**3. You cannot hide file identity through renaming**
Internal metadata survives renaming. Sysmon's masquerading detection proved this directly.

**4. Hash pivoting accelerates investigation**
Sysmon's automatic hash logging turned a suspicious process into a confirmed IOC in seconds via VirusTotal.

**5. A complete attack timeline is possible from 
endpoint logs alone**
Every stage of this attack from initial access to credential dumping was reconstructed purely from Sysmon events without any network monitoring tools.

---

## 🔗 Related Labs

- 🛡️ [Wazuh SIEM + Sysmon Integration](https://github.com/rohithbaggu56-dot/Wazuh-SIEM-SOC-Hands-On-Lab)
- 🏠 [Home SOC Lab](https://github.com/rohithbaggu56-dot/home-soc-lab)


### 📚 Learning Resource
Practiced based on TCM Security's Sysmon tutorial.All investigation, documentation and analysis is my own independent work.

---

🔗 **Navigation**
⬅️ [Back to Portfolio](https://github.com/rohithbaggu56-dot)
