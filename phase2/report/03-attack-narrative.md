# Attack Narrative

## Summary
The attack began with external reconnaissance against the DMZ web server, leading to a command injection foothold. By extracting a mounted SSH key and establishing a network tunnel, the team bypassed network segmentation to access the internal Analytics server, finally escalating to root via a sudo misconfiguration.

## Stage 1 — External Reconnaissance
The external enumeration of the DMZ identified a web server with Damn Vulnerable Web Application (DVWA) exposed on port 80 (evidence: `nmap-full.txt`).

## Stage 2 — Foothold via Command Injection
A command injection vulnerability was identified in the `/vulnerabilities/exec/` module. By bypassing CSRF protections, a reverse shell was established as the `www-data` user inside the Docker container (F-01).

## Stage 3 — Credential Discovery and Pivot
Enumeration of the container's filesystem revealed an SSH private key readable at `/var/www/.ssh/id_ed25519` (F-02). Because the Partner Portal VM held interfaces on both the DMZ and the internal segment and did not restrict forwarded traffic (F-03), the attacker was able to route packets from the DMZ side into the internal network using Ligolo-ng. This is the moment the segmentation control that was assumed to protect `10.20.30.0/24` ceased to apply.

## Stage 4 — Internal Foothold
The stolen SSH key was used to authenticate to the Analytics server (`10.20.30.40`) as the `rivermark` user. 

## Stage 5 — Privilege Escalation and Impact
Local enumeration revealed a critical misconfiguration in the sudoers file: `rivermark` could run `/usr/bin/vim` as root without a password (F-04). Using a standard GTFOBins technique, the team escalated to a root shell and accessed the confidential research drafts.