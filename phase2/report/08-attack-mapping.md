# MITRE ATT&CK Mapping

| ID | Technique | Where in Chain | Evidence |
|---|---|---|---|
| T1595.002 | Active Scanning: Vulnerability Scanning | Recon | nmap-full.txt |
| T1190 | Exploit Public-Facing Application | Foothold | exec-id-probe.html |
| T1059.004 | Command and Scripting Interpreter: Unix Shell | Foothold | reverse-shell-transcript.txt |
| T1552.004 | Unsecured Credentials: Private Keys | Credential Access | key-transfer.txt |
| T1572 | Protocol Tunneling | Lateral Movement | proxy-console.txt |
| T1021.004 | Remote Services: SSH | Lateral Movement | ssh-banner.txt |
| T1548.003 | Abuse Elevation Control Mechanism: Sudo | Privilege Escalation | sudo-l-mission5.txt |
| T1005 | Data from Local System | Collection | draft-alpha.md transcript |