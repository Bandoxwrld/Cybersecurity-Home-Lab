# Lab 01 — Nmap Network Reconnaissance

**Date:** March 31, 2026
**Analyst:** Sanaa Glasper
**Tool:** Nmap 7.98
**Platform:** Kali Linux 2026.1 on VirtualBox
**Target Range:** 127.0.0.1 and 10.0.2.0/24 (VirtualBox NAT)

---

## Objective

Perform a complete network reconnaissance exercise using Nmap inside a 
Kali Linux VM to practice host discovery, service enumeration, OS detection, 
and professional scan documentation — core skills for SOC Analyst and 
Network Technician roles.

---

## Lab Environment

| Component | Details |
|-----------|---------|
| Host Machine | MacBook Pro 13" A2289 — Intel Core i5, 8GB RAM |
| OS | macOS Sequoia 15.7.5 |
| VM Platform | VirtualBox 7.2.6 |
| Guest OS | Kali Linux 2026.1 |
| Nmap Version | 7.98 |
| Network | VirtualBox NAT — 10.0.2.0/24 |

---

## Step 1 — Environment Verification

**Command:**
```bash
nmap --version
```

**Result:** Nmap 7.98 confirmed installed on Kali Linux 2026.1.

Before beginning any reconnaissance activity, confirming the tool version 
is standard professional practice. This establishes the lab environment 
baseline and ensures the tool is current. Knowing the exact version matters 
because certain Nmap versions have known behavior differences in how they 
handle specific scan types, and version information is always included in 
professional reporting.

---

## Step 2 — Localhost Baseline Scan

**Command:**
```bash
nmap 127.0.0.1
```

**Result:** 0 open ports across all 1,000 scanned TCP ports. 
Host confirmed up with 0.000060s latency. Scan completed in 0.26 seconds.

Scanning the loopback address first establishes what a clean, secure machine 
looks like before comparing against other targets. Zero open ports means a 
minimal attack surface — exactly what a hardened machine should show. In a 
real SOC investigation, if this scan returned unexpected open ports on a 
workstation, those would immediately become points of interest.

---

## Step 3 — Service Version and OS Detection

**Command:**
```bash
sudo nmap -sV -O 127.0.0.1
```

**Result:** OS detection returned inconclusive — "too many fingerprints match." 
Network distance: 0 hops. Scan completed in 2.35 seconds.

The -sV flag probes open ports to identify software versions running on the 
target. The -O flag attempts OS fingerprinting by analyzing network responses. 
The inconclusive OS result is expected when scanning over loopback — the 
interface does not provide enough network behavior variation for reliable 
fingerprinting. In a real investigation, inconclusive OS detection on a live 
target can indicate a firewall, virtualization layer, or deliberate TTL 
manipulation — all worth documenting in an incident report.

---

## Step 4 — Network Interface Identification

**Command:**
```bash
ip addr
```

**Result:** Kali IP address confirmed as 10.0.2.15 on interface eth0.

Before scanning beyond the local machine, identifying your own position on 
the network is essential. This confirms the active network interface, the 
assigned IP address, and the network segment to be scanned. In a real SOC 
investigation this step establishes the analyst's own footprint before any 
outbound scanning begins.

---

## Step 5 — Host Discovery Across Network Range

**Command:**
```bash
sudo nmap -sn 10.0.2.0/24
```

**Result:** 3 live hosts discovered out of 256 addresses in 3.69 seconds.

| Host | Role |
|------|------|
| 10.0.2.2 | VirtualBox NAT Gateway |
| 10.0.2.3 | VirtualBox DNS Server |
| 10.0.2.15 | Kali Linux (self) |

The -sn flag performs host discovery without port scanning — mapping what 
is alive before deciding what to investigate further. This is the first phase 
of network reconnaissance in any real engagement. In a corporate environment, 
unexpected hosts appearing during this scan would be treated as anomalies 
requiring immediate investigation.

---

## Step 6 — Service and Script Scan of Gateway

**Command:**
```bash
sudo nmap -sV -sC 10.0.2.2
```

**Result:** 3 open ports discovered.

| Port | State | Service | Version |
|------|-------|---------|---------|
| 80 | open | HTTP/rtsp | AirTunes/870.14.1 |
| 7800 | open | HTTP/rtsp | AirTunes/870.14.1 |
| 8021 | open | tcpwrapped | Unknown |

The -sC flag runs Nmap's default scripting engine against discovered services, 
extracting additional protocol information and software fingerprints beyond 
basic port scanning.

### Notable Finding 1 — Unintended Service Exposure

Ports 80 and 7800 both returned HTTP 403 Forbidden responses with a server 
banner identifying as AirTunes/870.14.1 — Apple's AirPlay service running 
on the host MacBook bleeding through the VirtualBox NAT interface. This is 
a real-world example of unintended service exposure. The 403 response confirms 
the service is alive and rejecting unauthorized requests. In a corporate 
environment this finding would be documented and reviewed against the network 
baseline as a potential policy violation.

### Notable Finding 2 — tcpwrapped Port 8021

Port 8021 accepted the TCP connection but immediately disconnected before 
service identification. This is distinct from a closed port (rejects outright) 
or filtered port (drops silently). A tcpwrapped response indicates a service 
deliberately refusing to identify itself — worth flagging in any incident 
report as it can indicate TCP wrappers enforcing access control, a honeypot, 
or a service configured to only respond to pre-authorized clients.

---

## Step 7 — Saving Scan Output to File

**Command:**
```bash
sudo nmap -sV -sC 10.0.2.2 -oN ~/Desktop/gateway_scan.txt
```

**Result:** Output saved to gateway_scan.txt — verified with cat command.

Raw scan output saved as evidence artifact. Professional reconnaissance 
always produces saved output files — results read only on screen are not 
auditable or shareable. The -oN flag saves in normal human-readable format. 
This file is included in this repository as gateway_scan.txt.

---

## Summary of Findings

| Target | Open Ports | Key Finding |
|--------|-----------|-------------|
| 127.0.0.1 | 0 | Clean baseline — minimal attack surface |
| 10.0.2.0/24 | N/A | 3 live hosts mapped |
| 10.0.2.2 | 3 (80, 7800, 8021) | AirPlay exposure + tcpwrapped port |

---

## What I Learned

- Establishing a clean baseline before scanning other targets is essential 
  for identifying anomalies
- Real-world scans produce unexpected findings even in controlled lab 
  environments — the AirPlay exposure was not anticipated
- Professional documentation is as important as technical execution — saving 
  output files and writing up findings separates a skilled analyst from 
  someone who can only run a tool
- tcpwrapped responses require follow-up investigation — knowing what next 
  steps to take is a critical investigative skill
- Service banner information like AirTunes/870.14.1 can immediately identify 
  vendor, software, and potential known vulnerabilities

---

## Tools and Commands Reference

| Command | Purpose |
|---------|---------|
| `nmap --version` | Confirm tool version |
| `nmap 127.0.0.1` | Basic TCP port scan |
| `sudo nmap -sV -O` | Service version + OS detection |
| `ip addr` | Display network interfaces |
| `sudo nmap -sn 10.0.2.0/24` | Ping sweep — host discovery |
| `sudo nmap -sV -sC` | Service + NSE script scan |
| `nmap -oN filename` | Save output to file |
| `cat filename` | Read saved output file |

---

*Lab 1 of Cybersecurity Home Lab Portfolio | Sanaa Glasper | March 2026*
