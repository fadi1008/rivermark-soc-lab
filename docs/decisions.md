
---

## Corrected `docs/decisions.md` (full file)

```markdown
# Decision Log

## 2026-09-25 — Lab network design

**Decision:** Use VirtualBox Internal Network `rivermark-lan` (10.20.30.0/24) for all attack traffic between Kali and the portal.

**Reason:** Complete isolation from home LAN. No accidental scanning of real devices. Controlled environment.

**Temporary NAT:** Only rivermark-portal ever gets NAT, and only for updates. Kali never gets NAT. Never Bridged on the attacker VM.

**Alternatives considered:**
- Bridged — rejected. Puts Kali on the same L2 network as home devices.
- NAT Network — rejected. Provides internet by default, less isolation than Internal Network.
- Host-only — considered. Host joins lab LAN, which we do not want for attack traffic. Also, Host-only can reach the real host, so a spray from Kali there could hit real services.

**Consequences:** VMs cannot reach the internet unless the NAT cable is connected on the portal. Updates must be planned.

## 2026-09-25 — NAT procedure (corrected)

**Rule:** Only rivermark-portal gets NAT. Kali never does.

**Adapter layout on the portal:**
- Adapter 1: Internal rivermark-lan — static 10.20.30.20/24, no gateway. Always attached, cable connected.
- Adapter 2: NAT — present in VM config. Cable disconnected by default.

**Procedure:**
1. Snapshot before.
2. Tick Cable Connected on Adapter 2.
3. Boot. Confirm `ip route show default` shows NAT as default.
4. apt update / upgrade.
5. Untick Cable Connected on Adapter 2.
6. Confirm `ip route show default` no longer shows a default route.
7. Snapshot after: post-update-clean.

**Why cable disconnect, not adapter delete:** Deleting or hot-adding the NIC renames the interface (ens3 → ens18) and breaks netplan. Disconnecting the cable leaves the NIC in the VM config. Same interface name. No cable = interface down = no default route.

**Kali:** Adapter 1 Internal only. No Adapter 2. Ever.

**A missing default route is a security control.** After every change, `ip route show default` on the portal should show nothing — except during a documented update window.

## 2026-09-25 — Evidence path

**Rule:** Use a temporary Host-only adapter for evidence transfer. Never permanent shared folders.

**Procedure (Mission 2+):**
1. Power off portal. Add Host-only adapter.
2. Boot. SCP file from host.
3. Power off. Disconnect Host-only adapter (do not delete — same netplan reasoning).
4. Sanitize on host. Commit to Git.

**Note:** Not needed for Mission 2 (console paste is enough). Revisit when evidence transfer becomes the bottleneck. The realistic later pattern is a persistent Host-only on the portal only, acting as a management plane.

## 2026-09-25 — Answers to senior's five questions

**Q1:** Nobody assigns IPs on Internal Network. VirtualBox has no DHCP there. I must configure static IPs manually.

**Q2:** No, the host cannot ping 10.20.30.20 on Internal Network. The host is not on rivermark-lan. This is intentional isolation.

**Q3:** Traffic to 1.2.3.4 uses NAT (default route). Traffic to 10.20.30.20 uses Internal (directly connected). This is why Kali never gets NAT.

**Q4:** Kali 2 GB, portal 2 GB, host reserve 6–8 GB. Extended VMs (ws01 and siem) never run together.

**Q5:** Temporary Host-only adapter + SCP. Add, copy, disconnect. Sanitize before commit.

## 2026-09-25 — Kali role

**Decision:** Kali is the adversary, not the analyst workstation. The analyst workstation is rivermark-ws01 (Extended Lab). In Core Lab I dual-hat: attacks from Kali, logs read on the portal console.

## 2026-09-25 — Data classification

- Research drafts = Confidential
- Portal software = Internal
- Lab charter = Internal
- Lab logs/evidence = Internal (sanitize before commit)
- Snapshots = Internal (keep local, never commit VM images)

## 2026-09-25 — Matrix correction

**Correction:** NAT and NAT Network do not block outbound traffic. They hide you from the home LAN only. A password spray from a NAT-attached Kali can hit the internet. That is why Kali never gets NAT. Host-only can hit the real host (192.168.56.1), so it is also kept off Kali.

## 2026-09-25 — Portal RAM note

Portal at 2 GB is fine for Mission 2 (addressing + ping). Once MySQL + nginx exist, 2560–3072 MB is still Core Lab. Do not steal RAM from the host reserve to do it.