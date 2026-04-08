# Lab 05 - Simulated Attack and Detection

**Date:** April 8, 2026
**Analyst:** Sanaa Glasper
**Attack Tools:** Nmap 7.98, Nikto 2.6.0 — Kali Linux 2026.1
**Defense Tools:** Sysmon v15.20, Splunk Enterprise 10.2.2 — Windows 11
**Host (Target):** WIN-9FV73BAMF00

-

## Objective

Simulate a realistic attack sequence using Kali Linux offensive tools against a target network, then switch to the defensive perspective and detect equivalent attack behavior on a Windows 11 endpoint using Sysmon and Splunk. This lab ties together all previous labs into one complete offensive and defensive story.

-

## Lab Environment

| Component | Details |
|-----------|---------|
| Attack Machine | Kali Linux 2026.1 on VirtualBox |
| Target Network | VirtualBox NAT - 10.0.2.0/24 |
| Defense Machine | Windows 11 on UTM |
| SIEM | Splunk Enterprise 10.2.2 |
| EDR | Sysmon v15.20 with SwiftOnSecurity config |
| Host | WIN-9FV73BAMF00 |

-

## Phase 1 - Attack Simulation (Kali Linux)

### Attack 1 - Nmap Reconnaissance Scan

**Command:**
```bash
sudo nmap -sV -sC -O 10.0.2.2
```

**Result:** Host confirmed up. Ports 5000 and 7000 open running AirTunes/870.14.1 (Apple AirPlay). Service fingerprinting successful. OS detection inconclusive due to NAT layer.

A full service version and script scan was run against the gateway to simulate the reconnaissance phase of an attack. The -sV flag probes open ports for service versions, -sC runs default NSE scripts for additional information, and -O attempts OS fingerprinting. This is the standard first step an attacker takes after identifying a live target.

-

### Attack 2 - Aggressive Network Scan

**Command:**
```bash
sudo nmap -A -T4 10.0.2.0/24
```

**Result:** Full network range scanned. Same AirTunes service confirmed on gateway. MAC address 52:55:0A:00:02:02 identified. Network topology mapped.

The -A flag enables aggressive mode combining OS detection, version detection, script scanning and traceroute. The -T4 timing template speeds up the scan. This simulates an attacker mapping the full network range after initial access to understand the complete attack surface.

-

### Attack 3 - DNS Enumeration

**Command:**
```bash
nmap --script dns-brute 10.0.2.2
```

**Result:** DNS brute force attempted. Script returned "Can't guess domain of 10.0.2.2" - expected behavior against a raw IP without an associated domain. Ports 5000 (upnp) and 7000 (afs3-fileserver) confirmed open. MAC address confirmed.

DNS enumeration is a standard attacker technique to discover subdomains and internal hostnames. The inconclusive result against a raw IP is realistic - attackers encounter this limitation regularly in internal network environments and must adapt their approach.

-

### Attack 4 - Nikto Web Vulnerability Scan

**Command:**
```bash
nikto -h 10.0.2.2 -p 5000
```

**Result:** 8,206 requests made. 9 items reported. Scan duration: 361 seconds.

**Findings:**
- Server identified: AirTunes/870.14.1
- Uncommon headers: x-apple-processingtime, x-apple-requestreceivedtimestamp
- Missing: strict-transport-security header
- Missing: permissions-policy header
- Missing: referrer-policy header
- Missing: content-security-policy header
- Missing: x-content-type-options header
- X-Frame-Options deprecated
- X-Content-Type-Options not set - potential MIME type sniffing risk

Nikto probed the discovered web service for common vulnerabilities and misconfigurations. The missing security headers are real findings - in a penetration test report these would be documented as low to medium severity vulnerabilities requiring remediation.

-

## Phase 2 - Defense and Detection (Windows 11 + Sysmon + Splunk)

### Simulation 1 - System Reconnaissance

**Commands executed:**
```powershell
whoami
net user
net localgroup administrators
systeminfo
```

These commands simulate the first actions an attacker takes after gaining access - identifying the current user context, enumerating local accounts, checking group memberships, and profiling the system. All four commands were logged by Sysmon as Process Create events with full command line arguments.

-

### Simulation 2 - Network Reconnaissance

**Commands executed:**
```powershell
ipconfig /all
netstat -an
arp -a
route print
```

These commands map the network from the compromised endpoint - identifying all network interfaces, active connections, ARP cache entries, and routing tables. Attackers use this information to understand lateral movement opportunities and identify other hosts to target.

-

### Simulation 3 - Persistence Location Enumeration

**Commands executed:**
```powershell
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run
schtasks /query /fo LIST
```

These commands check the most common Windows persistence locations - registry run keys and scheduled tasks. Attackers enumerate these to understand what is already running at startup and to identify where to plant their own persistence mechanisms for maintaining access after reboot.

-

### Simulation 4 - Credential Enumeration

**Commands executed:**
```powershell
Get-LocalUser
Get-LocalGroup
net accounts
cmdkey /list
```

These commands enumerate local user accounts, group memberships, password policies, and stored credentials. Credential access is a critical phase of any attack - finding valid credentials enables privilege escalation and lateral movement across the network.

-

### Splunk Detection Results

**SPL Query:**
source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" "EventID>1<" "whoami" OR "net user" OR "netstat" OR "ipconfig" OR "reg query" OR "schtasks" OR "Get-LocalUser" OR "cmdkey"

**Result: 5 events detected**

Splunk returned 5 events matching the attack simulation commands. Expanding one event revealed the complete forensic picture Sysmon captured for the schtasks command:

| Field | Value |
|-------|-------|
| EventID | 1 (Process Create) |
| Computer | WIN-9FV73BAMF00 |
| User | WIN-9FV73BAMF00\Bando |
| Image | C:\Windows\system32\schtasks.exe |
| CommandLine | schtasks.exe /query /fo LIST |
| ParentImage | C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe |
| MD5 | 0EB0E5987DADBBCFAB61F46FC44ABE0 |
| SHA256 | B96AF3066F0B9DE8A460F77BAAE006C5372A266BE18C561DAA7C7AE8635D399D |

-

## Notable Findings

### Finding 1 - Complete Forensic Evidence Chain

Sysmon captured the exact command executed, the user who ran it, the parent process that launched it (PowerShell), the full file path of the executable, and cryptographic hashes proving exactly which version of the file ran. This is the complete evidence chain a SOC analyst needs to reconstruct what happened on an endpoint during an incident.

### Finding 2 - Missing Security Headers as Real Vulnerabilities

The Nikto scan identified five missing HTTP security headers on the AirTunes service. While this is Apple's AirPlay service and not a traditional web application, the finding demonstrates that any exposed web service should be evaluated for security header compliance. In a corporate environment these findings would be documented in a vulnerability report and tracked for remediation.

### Finding 3 - Attack Methodology Mirrors Real Threat Actors

The four-phase attack sequence used in this lab (reconnaissance, network mapping, persistence enumeration, credential access) directly mirrors the MITRE ATT&CK framework phases used by real threat actors. Understanding this methodology from both the offensive and defensive perspective is what enables SOC analysts to anticipate attacker behavior rather than just react to it.

-

## MITRE ATT&CK Mapping

| Technique | ID | Tool Used |
|-----------|-----|-----------|
| Network Service Scanning | T1046 | Nmap |
| OS Fingerprinting | T1082 | Nmap -O |
| DNS Brute Force | T1596.001 | Nmap dns-brute |
| Web Vulnerability Scanning | T1595.002 | Nikto |
| System Information Discovery | T1082 | systeminfo |
| Account Discovery | T1087 | net user, Get-LocalUser |
| Network Configuration Discovery | T1016 | ipconfig, netstat |
| Query Registry | T1012 | reg query |
| Scheduled Task Discovery | T1053 | schtasks |
| Credentials in Registry | T1552.002 | cmdkey /list |

-

## Summary

| Phase | Action | Result |
|-------|--------|--------|
| Attack 1 | Nmap reconnaissance | AirTunes/870.14.1 identified on ports 5000/7000 |
| Attack 2 | Aggressive network scan | Full topology mapped |
| Attack 3 | DNS enumeration | Inconclusive on raw IP - realistic result |
| Attack 4 | Nikto web scan | 9 findings including 5 missing security headers |
| Defense 1 | System recon simulation | Captured by Sysmon |
| Defense 2 | Network recon simulation | Captured by Sysmon |
| Defense 3 | Persistence enumeration | Captured by Sysmon |
| Defense 4 | Credential enumeration | Captured by Sysmon |
| Detection | Splunk SPL query | 5 attack simulation events detected |

-

## What I Learned

This lab demonstrated the complete offensive and defensive security cycle. Running attack tools from Kali and then switching to analyze the defensive telemetry in Splunk built genuine understanding of how attackers think and how defenders detect them. The MITRE ATT&CK mapping showed that the attack techniques used in this lab are not theoretical - they are the same techniques documented from real threat actor campaigns. Understanding the attack methodology is what enables a SOC analyst to write better detection rules, identify gaps in coverage, and prioritize which alerts matter most.

-

## Tools Reference

| Tool | Purpose |
|------|---------|
| `nmap -sV -sC -O` | Service version + script + OS detection |
| `nmap -A -T4` | Aggressive full scan |
| `nmap --script dns-brute` | DNS enumeration |
| `nikto -h target -p port` | Web vulnerability scanning |
| `whoami / net user / systeminfo` | System reconnaissance |
| `ipconfig / netstat / arp` | Network reconnaissance |
| `reg query / schtasks` | Persistence enumeration |
| `Get-LocalUser / cmdkey` | Credential enumeration |
| Splunk SPL query | Attack detection and correlation |

-

*Lab 5 of Cybersecurity Home Lab Portfolio - Sanaa Glasper - April 2026*
