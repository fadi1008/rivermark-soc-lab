# Rivermark SOC Lab — Lab Charter

**Author:** fadi bellalah
**Date:** 24-9-2026
**Version:** 1.0

---

## 1. Purpose

This lab exists to build and practice a security operations capability for a fictional small investment research firm, Rivermark Partners. The goal is not to collect tools or complete a CTF. The goal is to learn how to:

- Build and isolate a realistic small-company environment.
- Observe what normal looks like before introducing threats.
- Simulate controlled attacks inside the lab only.
- Detect, investigate, respond to, and harden against those attacks.
- Document work like a professional security analyst.

This is a defender's project. Offensive techniques are used only as evidence generators, strictly inside the isolated lab.

---

## 2. Company Context — Rivermark Partners

### What the firm does

Rivermark Partners is a small investment research firm that produces confidential equity research for professional clients. Analysts create reports containing financial analysis, models, and conclusions about companies before the reports are publicly released. The firm depends on keeping these research drafts confidential and preventing unauthorized modifications. A stolen or modified draft could expose sensitive information and damage the firm's reputation. The main security objective is therefore to protect unpublished research from unauthorized access, modification, or theft.

### Users

| User | Role | Access | Notes |
|---|---|---|---|
| sysadmin | System administrator | Root/sudo on Linux, local admin on Windows | Manages server, accounts, backups, configuration |
| dev_amine | Developer | SSH access to portal server | Maintains the research portal, handles deployments |
| dev_sarah | Developer | SSH access to portal server | Second developer, helps with deployments and app maintenance |
| analyst_yassine | Research analyst | Windows workstation, web portal login | Creates and accesses confidential research drafts |
| contractor_old | Former contractor | Disabled account | Should remain disabled. Represents an unnecessary account that could become a security risk. |

### Data

The main sensitive asset is the **research draft**.

A research draft may contain:
- Financial analysis
- Financial models
- Company information
- Investment conclusions
- Other non-public information

Unauthorized access could allow someone to copy or modify the research before publication. This could create financial, legal, regulatory, and reputational consequences for the firm.

**Where drafts live:** Research drafts live on the Linux portal server (in the web application and MySQL database) and, in later phases, on the analyst's Windows workstation in local files and browser sessions.

---

## 3. Lab Assets

The lab is built progressively.

### Core Lab — first phase

| Asset | OS | Role | IP | RAM |
|---|---|---|---|---|
| rivermark-portal | Ubuntu Server 24.04 | Web app, MySQL, SSH. Main target and crown jewel. | 10.20.30.20 | 2 GB |
| kali-offsec | Kali Linux | Adversary. Controlled security testing only. | 10.20.30.10 | 2 GB |

### Extended Lab — later (sequential, never simultaneous)

| Asset | OS | Role | IP | RAM |
|---|---|---|---|---|
| rivermark-ws01 | Windows 10/11 | Analyst workstation, local research files, browser sessions. | 10.20.30.30 | 4 GB |
| rivermark-siem | Ubuntu Server | Wazuh all-in-one. Log collection, detection, investigation. | 10.20.30.40 | 6 GB |

### Advanced Lab — much later

- Suricata or Zeek network sensor
- pfSense firewall VM
- Active Directory domain controller
- Additional network segments

## 4. Network Design

### Network decision (locked)

| Decision | Choice | Reason |
|---|---|---|
| Lab network type | VirtualBox Internal Network (`rivermark-lan`) | Complete isolation from home LAN. Host is not on the lab LAN. |
| Lab subnet | 10.20.30.0/24 | Private range, distinct from home 192.168.x. Mistyped scans fail closed. |
| Internet access | Temporary NAT on rivermark-portal only | Updates only. Kali never gets NAT. |
| Attacker networking | Never Bridged | Bridged puts Kali on the same L2 as home devices. Unacceptable. |

### IP allocation

| Host | IP | Role |
|---|---|---|
| kali-offsec | 10.20.30.10 | Adversary. Controlled attack simulation only. |
| rivermark-portal | 10.20.30.20 | Web app + MySQL + SSH. Crown jewel. |
| rivermark-ws01 | 10.20.30.30 | Analyst workstation (Extended Lab). |
| rivermark-siem | 10.20.30.40 | Wazuh SIEM (Extended Lab). |

### Update rule

Only rivermark-portal ever gets a NAT adapter. Kali never gets NAT.

- Adapter 1: Internal rivermark-lan — static 10.20.30.20/24, no gateway. Always attached.
- Adapter 2: NAT — present in VM config. Cable disconnected by default.

Procedure:
1. Snapshot before.
2. Tick Cable Connected on Adapter 2.
3. Boot. Confirm `ip route show default` shows NAT as default.
4. apt update / upgrade.
5. Untick Cable Connected on Adapter 2.
6. Confirm `ip route show default` no longer shows a default route.
7. Snapshot after: post-update-clean.

### Evidence transfer rule

Use a temporary Host-only adapter. Connect, SCP, disconnect. Never permanent shared folders. Sanitize on host before commit. Not needed for Mission 2.

## 5. Hardware Tier Decision

**Host machine specifications:**

| Component | Value |
|---|---|
| Host OS | Windows 11 |
| Total RAM | 16 GB |
| Free RAM (typical) | ~8 GB |
| CPU cores | 8 |
| Free disk | 150 GB |
| Virtualization enabled | Yes |

**Tier decision:** Core Lab. Extended is sequential, not simultaneous.

**Reason:** With 16 GB total and ~8 GB free, Core Lab (Kali 2 GB + portal 2 GB) leaves 6–8 GB for the host. Extended Lab VMs (Windows 4 GB, Wazuh 6 GB) cannot run together on this host.

**Core Lab RAM allocation:**

| VM | RAM |
|---|---|
| kali-offsec | 2 GB |
| rivermark-portal | 2 GB (can go to 2560–3072 MB if MySQL + nginx cause swapping) |
| Host reserve | 6–8 GB |

**Extended Lab RAM allocation (sequential):**

| VM | RAM |
|---|---|
| rivermark-ws01 | 4 GB |
| rivermark-siem | 6 GB |

**VMs that will never run at the same time:** rivermark-ws01 and rivermark-siem.

**Rule:** Do not steal RAM from the host reserve to enlarge any lab VM. If a VM needs more, accept the swap or move to a larger host.

## 6. Isolation — "Will Never" List

The following rules are non-negotiable for the entire project:

1. I will never use Bridged networking on the Kali attacker VM.
2. I will never scan, brute-force, or probe my home router, ISP, neighbor, employer, or any public internet host.
3. I will never run attacks outside the isolated `rivermark-lan` network.
4. I will never keep persistent internet access on lab VMs when it is not needed.
5. I will never skip a snapshot before an attack phase.
6. I will never store real credentials, real personal data, or real company data in the lab.
7. I will never commit unsanitized logs or screenshots containing real IPs or personal information to the public repo.
8. I will never treat a lab attack as "just practice" — if a command would be a crime outside the lab, it is still a crime outside the lab.

---

## 7. Repository Structure
rivermark-soc-lab/
README.md # Project overview
docs/
lab-charter.md # This document
architecture.md # Network and system diagrams
decisions.md # Dated decision log
evidence/ # Sanitized investigation notes and captures
detections/ # Custom detection rules (Wazuh, Sigma, etc.)
incidents/ # Incident reports
scripts/ # Automation scripts (Python, Bash, PowerShell)


---

## 8. Scope

### In scope

- Linux administration as a security function
- SSH, web app, and authentication security
- Log analysis (auth.log, syslog, web logs, auditd, later Windows/Sysmon)
- Network traffic analysis in an investigation context
- Detection engineering
- SOC-style alert triage and investigation
- Incident response on a small estate
- Controlled offensive techniques as evidence generators
- MITRE ATT&CK mapping
- Hardening based on observed incidents
- Lightweight automation
- Professional documentation

### Out of scope on purpose

- Exploit development
- Malware reverse engineering
- Cloud IAM at scale
- Active Directory attacks (Advanced Lab only)
- Enterprise-scale SOC architecture
- Nation-state threat modeling

---

## 9. Success Criteria

This lab is successful if, by the end, I can:

- Explain my network design and why it is isolated.
- Read raw logs and identify normal vs. suspicious activity.
- Detect at least three controlled attacks using rules I designed or tuned.
- Investigate an incident and produce a report with timeline, evidence, and impact.
- Contain and recover from a simulated incident.
- Harden the environment based on what I found.
- Automate at least one repetitive task.
- Present the project as a professional portfolio piece.

---

## 10. Decision Log

See `docs/decisions.md` for the full dated decision history.