# Phase 2 - Mission 5: Privilege Escalation and Impact

## Objective
Execute the planned privilege escalation path to obtain root access on the Analytics server and demonstrate impact by accessing the crown jewels.

## Exploitation
- **Vector:** `sudo NOPASSWD: /usr/bin/vim`
- **Command Executed:** `sudo vim -c ':!/bin/sh'`
- **Result:** Obtained an interactive shell as `uid=0(root)`.

## Impact Verification
- Successfully read the confidential file: `/srv/rivermark/drafts/draft-alpha.md` as root.
- This bypassed the `700` permissions that previously blocked the `rivermark` user, demonstrating full compromise of the data confidentiality objective.

## Conclusion
The internal Analytics server is fully compromised. Root access allows for complete control over the system, data exfiltration, and potential lateral movement to other internal assets.