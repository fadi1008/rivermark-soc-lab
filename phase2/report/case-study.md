# Case Study: Rivermark Phase 2 — DMZ to Internal Root

## Context
Rivermark Partners is a fictional investment research firm. Their critical asset is unpublished financial research data, stored on an isolated internal network. This project simulates a full red team engagement against a segmented network.

## Approach
I performed reconnaissance, exploited a public-facing web application to gain an initial foothold, bypassed network segmentation using a Layer 3 tunnel, and escalated privileges on the internal server to access the crown jewels. The engagement concluded with a formal, client-ready report featuring remediation advice and detection gap analysis.

## The Chain, in One Paragraph
An external attacker compromised a web application in the DMZ, gaining a low-privileged shell inside a Docker container. The attacker discovered an exposed SSH key mounted in the container. Using a network pivoting tool (Ligolo-ng), the attacker bypassed the network firewall to reach the isolated internal network. Using the stolen key, the attacker logged into the Analytics server as an unprivileged user and exploited a misconfigured `sudo vim` rule to escalate to root, gaining full access to the confidential research data.

## Key Findings
- **Critical:** Command Injection in the DMZ web application.
- **Critical:** Exposed SSH private key inside the DMZ container.
- **High (Chain Critical):** Privilege escalation to root via `sudo vim`.
- **Architectural:** Network segmentation controls are ineffective if they rely on credentials present on the compromised DMZ host.

## What I Learned
The most difficult part of this project was not the exploitation, but the reporting and analysis. Building the "Detection Gap Analysis" taught me to think like both an attacker and a SOC analyst. I learned that hardening changes your evidence (e.g., disabling password SSH broke my old detection rules), and that CSRF tokens can be bypassed programmatically with a little Python scripting.

## Skills Demonstrated
`Web App Exploitation` • `Command Injection` • `Docker Escape Analysis` • `SSH Key Handling` • `Ligolo-ng Network Pivoting` • `Linux Privilege Escalation (GTFOBins)` • `Report Writing` • `MITRE ATT&CK Mapping`