# F-04: Privilege Escalation via Sudo Vim

## Severity
High (CVSS 8.8)
*Note: Chain severity is Critical because this requires the prior exposed key (F-02).*
CVSS v3.1: AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H

## Affected component
Analytics Server (10.20.30.40) - `/etc/sudoers.d/rivermark-analytics`.

## Root Cause
The `rivermark` user is granted passwordless sudo execution of `/usr/bin/vim` (CWE-269: Improper Privilege Management). Because `vim` supports shell escapes (`:!`), this allows the user to spawn a root shell. This is a recurring problem because sudo allow-lists are easy to build from a man page and hard to audit without GTFOBins knowledge.

## Prerequisites
A valid SSH shell as the `rivermark` user (achieved via F-02).

## Evidence
See `phase2/report/evidence-redacted/sudo-l-mission5.txt`.

## Impact (demonstrated)
Successfully escalated to `uid=0(root)` and accessed the confidential research drafts located in `/srv/rivermark/drafts/`.

## Remediation
- **Symptom Fix:** Remove `vim` from the sudoers file.
- **Root Cause Fix:** Add a lint rule in the sudoers CI/CD step that fails if any binary in the GTFOBins shell-escape list appears in an allow-list.

## Retest procedure
1. `sudo -l -U rivermark` returns no NOPASSWD rule.
2. `sudo vim -c ':!/bin/sh'` fails with "a password is required".