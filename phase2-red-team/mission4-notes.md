# Phase 2 - Mission 4: Internal Foothold and Privilege-Escalation Enumeration

## Objective
Authenticate to the Analytics server as `rivermark` using the stolen SSH key, verify the pivot path, and enumerate privilege escalation vectors without exploiting them.

## Operational Note (Docker NAT + Ligolo Relay)
Traffic reaching the Analytics server originated from the Partner Portal host's internal interface (`10.20.30.30`), confirming the segmentation failure.

## Chain of Custody (Key Exfiltration)
- Extracted the private key (`/var/www/.ssh/id_ed25519`) from the container using `/dev/tcp`.
- **Private Key Fingerprint (Kali):** `SHA256:...` (fill in)
- **Public Key Fingerprint (Container):** `SHA256:...` (fill in)
- Verified both fingerprints matched before proceeding.

## Foothold Verification
- `id` confirms access as `uid=1000(rivermark)`.
- `$SSH_CLIENT` shows `10.20.30.30`, confirming the reverse proxy path via the DMZ container.

## Intended Escalation Path (UNEXECUTED)
**Vector:** `sudo NOPASSWD: /usr/bin/vim`
- **Preconditions Verified:** Yes. `sudo -l` shows the rule.
- **Predicted Execution:** Running `sudo vim -c ':!/bin/sh'` will drop a root shell.
- **Impact if confirmed:** Root can bypass file permissions and read `/srv/rivermark/drafts/`.
- **Status:** Confirmed vulnerability exists, but NOT exploited (per mission rules).

## Impact Target Verification
- Running `ls -la /srv/rivermark/drafts/` as `rivermark` returned **Permission Denied**.
- Root access is required to read the crown jewels.