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



## 2026-09-25 — Ubuntu Server ISO verification

**ISO filename:** ubuntu-24.04.5-live-server-amd64.iso
**Source:** https://releases.ubuntu.com/24.04.5/
**My SHA256 (computed on my disk):** 97f3d7ffb032c3eb3b23d2c8be9cc76e60c2c1f2c0146ba5ba9fe01cafae0fd8
**Canonical SHA256 (published):** 97f3d7ffb032c3eb3b23d2c8be9cc76e60c2c1f2c0146ba5ba9fe01cafae0fd8
**Match:** yes
**Method:** Get-FileHash -Algorithm SHA256 (PowerShell)
**Date verified:** 2026-09-25








## 2026-10-09 — Mission 2 Task E: Isolation proof

**Kali → portal (10.20.30.20):** Success — 4/4 packets
**Portal → Kali (10.20.30.10):** Success — 4/4 packets
**Kali → home gateway (172.16.48.1):** Network unreachable
**Kali → 1.1.1.1:** Network unreachable
**Portal → home gateway:** Network unreachable
**Portal → 1.1.1.1:** Network unreachable
**Windows → 10.20.30.20:** 100% loss
**Windows → 192.168.56.20:** Success (management plane)

**Three-sentence interpretation:**
- Lab↔lab ping success proves both VMs are on the same Internal Network `rivermark-lan` and can exchange IP traffic directly, the prerequisite for all later attack and detection work.
- Failure to the home gateway proves neither VM has a default route off the lab, so no accidental scan or exploit can reach the home network.
- Host failure to `10.20.30.20` proves the Windows host is not on the lab LAN, so the lab is isolated from the host's production network.

**What a successful 1.1.1.1 ping would have meant:**
It would have meant a default route existed on one of the VMs (likely via a left-connected NAT cable or a wrong netplan gateway), which breaks isolation.

## 2026-10-09 — Task E discovery checkpoint answers

**D1 — Portal had no IPv4 after first boot. Broken OS?**
No. It was exactly the network I asked for: Internal Network with no DHCP and no static config yet.

**D2 — apt update fails on the portal. Incident or test?**
A passed test. No default route means no route off the lab.

**D3 — Did connecting NAT "just to see" grow a default route?**
Yes — via 10.0.3.2 (VirtualBox NAT gateway). I checked `ip route` before and after. Without looking, I would have missed a route that breaks isolation.

## 2026-10-09 — Clock decision

Portal and Kali both show UTC. Skew is ~20 seconds between them. Acceptable for Mission 2. NTP not configured (no persistent internet). Revisit when Wazuh is added.

## 2026-10-09 — Boot time fix

**Problem:** Portal took ~3 minutes to boot.
**Cause:** systemd-networkd-wait-online.service waited 2 minutes for a network state that never arrives. cloud-init ran for 205 seconds trying to reach a metadata service.
**Fix:**
- Systemd drop-in override for systemd-networkd-wait-online.service (ExecStart=/bin/true).
- Did NOT `systemctl disable` cloud-init services — that hangs boot.
**Lesson:** Masking a service is not enough when netplan regenerates its config in `/run/systemd/system/`. Use a drop-in override in `/etc/systemd/system/`.

## 2026-10-09 — Mission 2 reflection

**What surprised me most:** How much of a realistic network can be built using only configuration. VirtualBox settings (adapter type, Internal Network name, cable state) plus netplan/NMCLI produce a segmented, isolated, addressable environment. The same principles that govern real data centers — segmentation, default routes, ARP, gateway behavior — appear in miniature. Network security is about correctly configuring and verifying the basics, not about fancy tools.

## 2026-10-09 — Snapshots

- Portal: `m2-portal-static-ip` — early state with static IP
- Portal: `m2-portal-boot-fixed` — current best state, boot fixed, Host-only management plane
- Kali: `m2-on-rivermark-lan` — fully configured attacker VM

For Mission 3 onwards, restore from `m2-portal-boot-fixed` and `m2-on-rivermark-lan`.