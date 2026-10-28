# Methodology and Limitations

## Methodology
The assessment followed a standard penetration testing methodology:
1. **Reconnaissance:** Active scanning and service enumeration.
2. **Exploitation:** Exploiting web application vulnerabilities (Command Injection).
3. **Pivoting:** Bypassing network segmentation using Ligolo-ng.
4. **Privilege Escalation:** Enumerating and exploiting local misconfigurations.

## Tools Used
- **Nmap** (Scanning)
- **Custom Python Script** (CSRF bypass & payload delivery)
- **Ligolo-ng** (Network Pivoting)
- **GTFOBins** (Privilege Escalation Research)

## Limitations & Unexplored Hypotheses
- **Unexplored Hypotheses:** Optional secondary vectors (such as testing for weak passwords on the Analytics server or evaluating IDOR on DVWA) were not pursued. They remain as untested hypotheses for a follow-on engagement.
- **Log Retention:** The container's stdout was the primary source of DVWA logs; a production deployment should route container logs to a central system with retention >= 90 days.