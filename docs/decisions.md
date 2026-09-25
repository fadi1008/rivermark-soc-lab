# Decision Log

## 2026-09-24 — Lab network design

**Decision:** Use VirtualBox Internal Network `rivermark-lan` (10.20.30.0/24) for all attack traffic between Kali and the portal.

**Reason:** Complete isolation from home LAN. No accidental scanning of real devices. Controlled environment.

**Temporary NAT:** Only rivermark-portal ever gets NAT, and only for updates. Kali never gets NAT. Never Bridged on the attacker VM.

**Alternatives considered:**
- Bridged — rejected. Puts Kali on the same L2 network as home devices.
- NAT Network — rejected. Provides internet by default, less isolation than Internal Network.
- Host-only — considered. Host joins lab LAN, which we do not want for attack traffic.

**Consequences:** VMs cannot reach the internet unless a temporary NAT adapter is added to the portal. Updates must be planned.

## 2026-09-24 — NAT procedure

**Rule:** Only rivermark-portal gets NAT. Kali never does.

**Procedure:**
1. Snapshot before.
2. Power off. Add NAT adapter.
3. Boot. Confirm `ip route`.
4. Run updates.
5. Power off. Remove NAT adapter entirely (not just disconnect cable).
6. Boot. Confirm no default route via NAT.
7. Snapshot after: `post-update-clean`.

## 2026-09-24 — Evidence path

**Rule:** Use a temporary Host-only adapter for evidence transfer. Never permanent shared folders.

**Procedure:**
1. Power off portal. Add Host-only adapter.
2. Boot. SCP file from host.
3. Power off. Remove Host-only adapter.
4. Sanitize on host. Commit to Git.

## 2026-09-24 — Answers to senior's five questions

**Q1:** Nobody assigns IPs on Internal Network. VirtualBox has no DHCP there. I must configure static IPs manually.

**Q2:** No, the host cannot ping 10.20.30.20 on Internal Network. The host is not on rivermark-lan. This is intentional isolation.

**Q3:** Traffic to 1.2.3.4 uses NAT (default route). Traffic to 10.20.30.20 uses Internal (directly connected). This is why Kali never gets NAT.

**Q4:** Kali 2 GB, portal 2 GB, host reserve 6–8 GB. Extended VMs (ws01 and siem) never run together.

**Q5:** Temporary Host-only adapter + SCP. Add, copy, remove. Sanitize before commit.

## 2026-09-24 — Kali role

**Decision:** Kali is the adversary, not the analyst workstation. The analyst workstation is rivermark-ws01 (Extended Lab). In Core Lab I dual-hat: attacks from Kali, logs read on portal.

## 2026-09-24 — Data classification

- Research drafts = Confidential
- Portal software = Internal
- Lab charter = Internal
- Lab logs/evidence = Internal (sanitize before commit)