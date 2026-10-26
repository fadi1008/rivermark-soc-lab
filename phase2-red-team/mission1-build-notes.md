# Phase 2: Mission 1 — DMZ and Internal Segment Build

## Scope & Architecture Changes
- Kali has been moved strictly to the DMZ (`10.20.40.10`). 
- The original `rivermark-portal` is powered off and out of scope for Phase 2.
- Management plane remains on Host-Only (`192.168.56.10`).
- The only path from the DMZ to the internal network (`10.20.30.0/24`) is through the Partner Portal.

## Evidence of Isolation
**From Kali (DMZ):**
- `ip route | grep 10.20.30` → returns nothing.
- `ping 10.20.30.40` → 100% packet loss (Destination Host Unreachable).
- `nc -vz 10.20.30.40 22` → failed (timeout).

**From Partner Portal (Dual-homed):**
- `ping 10.20.30.40` → success.
- `nc -vz 10.20.30.40 22` → succeeded.

## DVWA Setup
- DVWA running on Docker on the Partner Portal.
- Exposed on port 80 (`10.20.40.20`).
- Security level set to Low.

## Lateral Movement Preparation
- Generated Ed25519 keypair on Portal at `/var/www/.ssh/id_ed25519`.
- Added public key to `/home/rivermark/.ssh/authorized_keys` on Analytics.
- SSH from Portal as `www-data` successfully executes commands as `rivermark` on Analytics (Test 3 passed).

## Privilege Escalation Configuration
- Created `/etc/sudoers.d/rivermark-analytics`.
- Rule: `rivermark ALL=(root) NOPASSWD: /usr/bin/vim`.
- `visudo -cf` returns "parsed OK".

## Firewalls (UFW)
- Both VMs have NAT adapters removed and no default routes.
- UFW enabled on both.
- Portal allows port 80 from DMZ and port 22 from Internal & Management.
- Analytics allows port 22 from Internal & Management.