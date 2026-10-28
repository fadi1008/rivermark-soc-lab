# Attack Timeline

*Note: All timestamps reconstructed from sequential execution order during the engagement.*

| UTC Timestamp | Host | Event | Evidence Source | ATT&CK Technique |
|---|---|---|---|---|
| 2026-10-28T14:00:00Z | Kali | nmap scan of 10.20.40.20 | nmap-full.txt | T1595.002 |
| 2026-10-28T14:05:00Z | Kali | Reconnaissance of DVWA | kali-history.txt | T1595 |
| 2026-10-28T14:15:00Z | Portal | DVWA login as admin | nginx access log | T1078 |
| 2026-10-28T14:25:00Z | Portal | Command Injection POST `/vulnerabilities/exec/` | nginx access log | T1190 |
| 2026-10-28T14:26:00Z | Portal | Reverse shell established to 10.20.40.10:4444 | Kali nc listener | T1059.004 |
| 2026-10-28T14:30:00Z | Portal | Key exfiltration (`/var/www/.ssh/id_ed25519`) | key-transfer.txt | T1552.004 |
| 2026-10-28T14:35:00Z | Portal | Ligolo Tunnel established (port 11601) | proxy-console.txt | T1572 |
| 2026-10-28T14:40:00Z | Kali | Route added for 10.20.30.0/24 | kali-history.txt | T1090 |
| 2026-10-28T14:45:00Z | Analytics | SSH Login as rivermark from 10.20.30.30 | auth.log | T1021.004 |
| 2026-10-28T14:50:00Z | Analytics | Enumeration (`sudo -l`, SUID, etc.) | Enum output | T1087 |
| 2026-10-28T14:55:00Z | Analytics | Privilege Escalation: `sudo vim -c ':!/bin/sh'` | auth.log | T1548.003 |
| 2026-10-28T14:56:00Z | Analytics | Root read of `/srv/rivermark/drafts/draft-alpha.md` | Terminal Transcript | T1005 |