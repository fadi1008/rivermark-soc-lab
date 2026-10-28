# F-01: Command Injection in DVWA

## Severity
Critical (CVSS 9.8)
CVSS v3.1: AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H

## Affected component
Partner Portal (10.20.40.20) - DVWA `/vulnerabilities/exec/` module.

## Root Cause
The application passes the `ip` POST parameter directly to the operating system's shell without any sanitization or input validation. This allows an attacker to append arbitrary system commands to the ping command.

## Prerequisites for exploitation
Network access to the DMZ (10.20.40.0/24).

## Evidence
See `phase2/evidence/mission2/exec-id-probe.html`.

## Impact (demonstrated)
An unauthenticated attacker (once logged into DVWA) can execute arbitrary shell commands as the `www-data` user inside the container, leading to a full reverse shell.

## Remediation
Implement strict input validation (allow only valid IP addresses). Avoid using shell execution functions (like `os.system` or `shell_exec`) and use safer language-specific APIs instead.