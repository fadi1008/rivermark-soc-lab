# F-04: Privilege Escalation via Sudo Vim

## Severity
High (CVSS 8.8)
*Note: Chain severity is Critical due to the prior exposed key (F-02) required to reach this stage.*
CVSS v3.1: AV:L/AC:L/PR:L/UI:N/S:U/C:H/I:H/A:H

## Affected component
Analytics Server (10.20.30.40) - `/etc/sudoers.d/rivermark-analytics`.

## Root Cause
The `rivermark` user is granted passwordless sudo execution of `/usr/bin/vim`. Because `vim` supports shell escapes (`:!`), this allows the user to spawn a root shell.

## Prerequisites for exploitation
A valid SSH shell as the `rivermark` user (achieved via F-02).

## Evidence
See `phase2/evidence/mission5/sudo-l-mission5.txt` and the terminal transcript.

## Impact (demonstrated)
Successfully escalated to `uid=0(root)` and accessed the confidential research drafts located in `/srv/rivermark/drafts/`.

## Remediation
Remove `vim` from the sudoers file. If users need to edit specific files with elevated privileges, use `sudoedit` and restrict the specific files they are allowed to edit.