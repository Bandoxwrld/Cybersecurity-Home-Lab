# Lab 04 - Sysmon Endpoint Telemetry

**Date:** April 7, 2026
**Analyst:** Sanaa Glasper
**Tool:** Sysmon v15.20
**Config:** SwiftOnSecurity sysmonconfig-export.xml
**Platform:** Windows 11 on UTM
**Host:** WIN-9FV73BAMF00
**Events Generated:** 696 in Event Viewer | 2,664 Process Create events in Splunk

-

## Objective

Install and configure Microsoft Sysmon on a Windows 11 VM using the SwiftOnSecurity community ruleset, verify endpoint telemetry in Event Viewer, forward Sysmon logs into Splunk, and demonstrate the dramatic difference in visibility between standard Windows Event Logs and Sysmon telemetry.

-

## Lab Environment

| Component | Details |
|-----------|---------|
| Host Machine | MacBook Pro 13" A2289 - Intel Core i5, 8GB RAM |
| OS | macOS Sequoia 15.7.5 |
| VM Platform | UTM |
| Guest OS | Windows 11 |
| Sysmon Version | v15.20 |
| Configuration | SwiftOnSecurity sysmonconfig-export.xml |
| Splunk Version | 10.2.2 |

-

## Step 1 - Sysmon Download and Extraction

**Commands used:**
```powershell
Expand-Archive -Path Sysmon.zip -DestinationPath Sysmon
```

Sysmon was downloaded from the official Microsoft Sysinternals page. The package contained three executables - Sysmon (32-bit), Sysmon64 (64-bit Intel/AMD), and Sysmon64a (64-bit ARM). Sysmon64 was selected since the Windows 11 VM runs on an Intel Mac via UTM. The SwiftOnSecurity configuration file (sysmonconfig-export.xml) was downloaded simultaneously from github.com/SwiftOnSecurity/sysmon-config - the most widely deployed Sysmon ruleset in the industry, used in real enterprise SOC environments worldwide.

-

## Step 2 - Sysmon Installation with Configuration

**Command:**
```powershell
.\Sysmon64.exe -accepteula -i C:\Users\Bando\Desktop\sysmon-config-master\sysmonconfig-export.xml
```

**Result:**
- Configuration file validated against schema 4.50
- Sysmon schema version: 4.91
- Sysmon64 installed
- SysmonDrv kernel driver installed
- SysmonDrv started
- Sysmon64 started

Sysmon was installed from an Administrator PowerShell session. The -accepteula flag silently accepted the license and -i applied the configuration at install time. The kernel driver (SysmonDrv) is what gives Sysmon its deep visibility - it operates at the kernel level intercepting system calls before they complete, capturing telemetry that user-mode tools cannot see and that is significantly harder for malware to evade.

-

## Step 3 - Event Viewer Verification

**Location:** Applications and Services Logs - Microsoft - Windows - Sysmon - Operational

**Result:** 696 Process Create events captured immediately after installation.

Event Viewer confirmed Sysmon was generating events immediately after install. All events showed Event ID 1 (Process Create) with full process details including ProcessGuid, UtcTime, and parent process information. This confirmed Sysmon was operating correctly at the kernel level and capturing all process execution activity in real time.

-

## Step 4 - Sysmon Log Forwarding to Splunk

**Config file created:** C:\Program Files\Splunk\etc\system\local\inputs.conf

**Configuration added:**
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
index = default
disabled = false
renderXml = true
source = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational

**Restart command:**
```powershell
.\splunk.exe restart
```

**Result:** All preliminary checks passed. All installed files intact. Splunkd started successfully.

A custom inputs.conf file was created to instruct Splunk to monitor the Sysmon event log channel in XML format for maximum field extraction. Splunk was restarted to load the new configuration.

-

## Step 5 - Sysmon Data Verification in Splunk

**Search:** `source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"`

**Result:** 497 total Sysmon events confirmed flowing into Splunk.

**Process Create search:** `source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational" "EventID>1<"`

**Result:** 2,664 Process Create events with full telemetry.

**Fields available on every event:**
- MD5 hash
- SHA256 hash
- IMPHASH (import hash for malware family identification)
- Process name and full path
- Parent process name and path
- Full command line arguments
- ProcessID and ProcessGuid
- UTC timestamp

-

## Notable Findings

### Finding 1 - Cryptographic Hashes Enable Threat Intelligence Integration

Every Sysmon Process Create event contains MD5, SHA256, and IMPHASH hashes of the executed file. This enables instant threat intelligence lookups - a SOC analyst can copy any hash from a suspicious process and check it against VirusTotal or any threat intelligence platform in seconds. The IMPHASH is particularly powerful because it identifies malware families even when the file has been recompiled to evade hash-based detection. Standard Windows Event Logs provide none of this.

### Finding 2 - Sysmon vs Standard Windows Event Logs

| Capability | Standard Event Logs | Sysmon |
|-----------|-------------------|--------|
| Process execution | Basic name only | Full path, hashes, parent, cmdline |
| File hashes | None | MD5, SHA256, IMPHASH |
| Network connections | Limited | Full source/dest IP, port, process |
| DNS queries | None | Full query and response |
| Registry changes | Limited | Full key/value with process context |
| Process injection | None | CreateRemoteThread detection |
| Tamper resistance | User-mode (bypassable) | Kernel driver (harder to evade) |

-

## Key Sysmon Event IDs

| Event ID | Description |
|----------|-------------|
| 1 | Process Create - full details with hashes |
| 2 | File creation time changed - timestomping indicator |
| 3 | Network connection - outbound connections |
| 5 | Process terminated |
| 7 | Image loaded - DLL loaded into process |
| 8 | CreateRemoteThread - process injection indicator |
| 10 | Process accessed - credential dumping indicator |
| 11 | File created - file system activity |
| 12/13 | Registry events - persistence indicator |
| 22 | DNS query - C2 beaconing detection |

-

## Summary

| Step | Result |
|------|--------|
| Sysmon installed | v15.20 with SysmonDrv kernel driver |
| Event Viewer | 696 Process Create events captured |
| Splunk forwarding | inputs.conf configured, Splunk restarted |
| Events in Splunk | 497 total Sysmon events |
| Process Creates | 2,664 events with full hash telemetry |

-

## What I Learned

This lab demonstrated why Sysmon is considered essential infrastructure for any mature SOC environment. The difference between standard Windows Event Log telemetry and Sysmon telemetry showed how much visibility a SOC analyst gains by deploying Sysmon. The SwiftOnSecurity configuration provides a production-ready ruleset that balances comprehensive coverage with noise reduction. Understanding how to install, configure, and ingest Sysmon into a SIEM is a directly employable skill that appears in SOC job postings regularly.

-

## Tools and Commands Reference

| Command | Purpose |
|---------|---------|
| `Expand-Archive -Path Sysmon.zip -DestinationPath Sysmon` | Extract Sysmon ZIP |
| `.\\Sysmon64.exe -accepteula -i config.xml` | Install Sysmon with config |
| `eventvwr.msc` | Open Event Viewer |
| `.\\splunk.exe restart` | Restart Splunk |
| `source="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"` | Search Sysmon in Splunk |

-

*Lab 4 of Cybersecurity Home Lab Portfolio - Sanaa Glasper - April 2026*
