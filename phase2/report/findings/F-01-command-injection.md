# F-01: Command Injection in DVWA

## Severity
Critical (CVSS 9.8)
CVSS v3.1: AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H

## Affected component
Partner Portal (10.20.40.20) - DVWA `/vulnerabilities/exec/` module.

## Root Cause
The application passes user-supplied input (`ip` parameter) to the operating system's shell without sanitization. This is a design flaw (CWE-78: Improper Neutralization of Special Elements used in an OS Command). This class of issue is common because developers often use shell commands for convenience without realizing the security implications.

## Prerequisites
Network access to the DMZ (10.20.40.0/24).

## Evidence
See `phase2/report/evidence-redacted/exec-id-probe.html`.

## Impact (demonstrated)
Achieved unauthenticated command execution as `www-data` inside the container, leading to a full reverse shell.

## Remediation
- **Symptom Fix:** Use `escapeshellarg()` and strict input allow-listing.
- **Root Cause Fix:** Avoid shell execution functions entirely; use language-specific APIs.

## Retest procedure
1. Submit `127.0.0.1; id` to the exec module.
2. Confirm the response does not contain `uid=33(www-data)`.