# Detection Gap Analysis

| Attack Step | Telemetry Available | Phase 1 Detection | Would it have caught this? | Gap / New Detection Opportunity |
|---|---|---|---|---|
| Command Injection POST | nginx access log | Web auth detection | No. Phase 1 rules only caught brute force (200 vs 302). | Add rule: alert on `/vulnerabilities/*` requests containing `;`, `\|`, `&&`, `$(`. |
| Reverse shell to 10.20.40.10:4444 | UFW log (if enabled) | None | No. | Add rule: monitor `ss -tnp` on containers for outbound connections on non-standard ports. |
| Exposed SSH Key | None (no auditd) | None | No. | **Critical Gap:** Implement `auditd` to monitor reads on `/var/www/.ssh/`. |
| Ligolo Tunnel | UFW log | None | No. | Add rule: alert on long-lived outbound connections from containers. |
| SSH from Kali to Analytics | Analytics `auth.log` | SSH auth detection (Phase 1) | **Yes.** Phase 1 rules would catch a successful login from a new source IP. | None. Phase 1 worked here. |
| Sudo Vim | Analytics `auth.log` | None | No. | Add rule: alert on `sudo` invocations of GTFOBins binaries (`vim`, `less`, `awk`, `find`). |

## Priority Additions
1. **Auditd on SSH keys:** The key access was silent.
2. **Sudo GTFOBins alert:** The privilege escalation is trivially detectable if you know the binary.
3. **Web Command Injection WAF/Rule:** Catch the malicious characters in the POST request before it executes.