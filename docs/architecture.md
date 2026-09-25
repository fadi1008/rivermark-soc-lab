# Rivermark SOC Lab — Architecture

**Author:** fadi bellalah
**Date:** 24-9-2026
**Version:** 1.0

---

## 1. Network Diagram
YOUR WINDOWS 11 HOST
(not on rivermark-lan)
|
VirtualBox
|
+---------------+---------------+
| |
[Internal Network] [Host-only adapter]
rivermark-lan (temporary, for evidence)
10.20.30.0/24 |
| |
+---------+---------+ |
| | |
[kali-offsec] [rivermark-portal] |
10.20.30.10 10.20.30.20 |
(adversary) (web + MySQL + SSH) |
| |
+-------------------------+
(SCP evidence out)


**Extended Lab (later):**

rivermark-lan — 10.20.30.0/24
|
+---------------+---------------+
| | |
[kali-offsec] [rivermark-portal] [rivermark-ws01]
10.20.30.10 10.20.30.20 10.20.30.30
(Windows analyst
workstation)

[rivermark-siem]
10.20.30.40
(Wazuh — runs sequentially,
never with rivermark-ws01)


**Advanced Lab (much later):**

- Suricata or Zeek network sensor
- pfSense firewall VM
- Active Directory domain controller
- Additional network segments

---

## 2. Network Design Decision

| Decision | Choice | Reason |
|---|---|---|
| Lab network type | VirtualBox Internal Network (`rivermark-lan`) | Complete isolation from home LAN. The host is not on the lab LAN. |
| Lab subnet | 10.20.30.0/24 | Private range, distinct from typical home 192.168.x. Mistyped scans fail closed. |
| Internet access | Temporary NAT on `rivermark-portal` only | Updates only. Kali never gets NAT. |
| Attacker networking | Never Bridged | Bridged puts Kali on the same L2 network as home devices. Unacceptable. |

**Key insight:** Internal Network is the only VirtualBox mode where the host is **not** on the lab LAN. The host cannot accidentally reach the lab, and the lab cannot reach the host. This is why it is the safest mode for attack traffic.

**Trade-off:** No internet, no host access. Both must be solved deliberately.

---

## 3. Network-Mode Matrix

Answers to the five required questions for each VirtualBox network mode.

| Mode | Can VM reach internet? | Can VM reach other VMs? | Can VM reach host? | Can VM reach home Wi-Fi devices? | Could a password spray from Kali hit something that is not yours? |
|---|---|---|---|---|---|
| **Bridged** | Yes | Yes (same L2) | Yes | **Yes** | **Yes — dangerous** |
| **NAT** | Yes | No (unless port-forward) | No | No | No |
| **NAT Network** | Yes | Yes (same NAT network) | Limited (port-forward) | No | No |
| **Host-only** | No | Yes (same host-only network) | Yes (host has adapter) | No | No |
| **Internal Network** | No | Yes (same internal network) | **No** | No | No |

**Why Internal Network was chosen:**

- The host is not on the lab LAN. This prevents the host from accidentally interacting with attack traffic.
- No internet by default. A mistyped scan cannot leave the lab.
- Isolation is enforced at the hypervisor level.

**Why Bridged was rejected:**

- Bridged places Kali on the same Layer 2 network as home devices (router, printer, phones, other PCs).
- A password spray against `192.168.1.0/24` from Kali would target real devices.
- This is unacceptable and potentially illegal.

---

## 4. IP Allocation

| Host | IP | Subnet | Role |
|---|---|---|---|
| kali-offsec | 10.20.30.10 | rivermark-lan | Adversary. Controlled attack simulation only. |
| rivermark-portal | 10.20.30.20 | rivermark-lan | Web app + MySQL + SSH. Crown jewel. |
| rivermark-ws01 | 10.20.30.30 | rivermark-lan | Analyst workstation (Extended Lab). |
| rivermark-siem | 10.20.30.40 | rivermark-lan | Wazuh SIEM (Extended Lab). |

**Why these addresses:**

- `.10` for the attacker keeps it memorable and first.
- `.20` for the crown jewel.
- `.30` for the analyst workstation.
- `.40` for the SIEM.
- The `10.20.30.0/24` range is distinct from typical home networks (192.168.x.x), so a mistyped scan fails closed.

---

## 5. Update Rule — NAT Procedure

**Rule:** Only `rivermark-portal` ever gets a temporary NAT adapter. Kali never gets NAT.

**Why:** Kali is the attacker. If Kali has NAT, a mistyped scan could leave the lab via the default route. The portal only needs NAT to fetch updates. The portal is not running attack tools.

**Procedure:**

1. Snapshot the VM before adding NAT.
2. Power off the VM.
3. Add a second adapter: NAT.
4. Boot. Confirm `ip route show default` shows NAT as the default route.
5. Run updates: `sudo apt update && sudo apt upgrade -y`.
6. Power off.
7. **Remove the NAT adapter entirely** (do not just disconnect the cable — deleting is cleaner for netplan).
8. Boot. Confirm `ip route show default` no longer shows a default route via NAT.
9. Snapshot again: `post-update-clean`.

**What to watch for:**

- Hot-adding or removing NICs changes interface names (`ens3` → `ens7`). This breaks static netplan config.
- Having a NIC and disconnecting the virtual cable is not the same as deleting the NIC. One wrecks your addressing; one is a control.
- Always power off before adding or removing adapters.

---

## 6. Evidence Transfer Rule

**Rule:** Use a temporary Host-only adapter for evidence transfer. Never use permanent shared folders.

**Why:** Internal Network means the host is not on `rivermark-lan`. This is a feature. But you need a way to move sanitized logs into the Git repo without bridged networking. Shared folders give the VM a permanent writable view of your real user profile — that is a trust risk.

**Procedure:**

1. Power off `rivermark-portal`.
2. Add a Host-only adapter (second NIC).
3. Boot. The host can now reach the portal on a separate host-only subnet (e.g., `192.168.56.0/24`).
4. From the host, `scp` the sanitized log file out.
5. Power off. Remove the Host-only adapter.
6. Sanitize the file on the host (remove real IPs, usernames if needed).
7. Commit to Git.

**Never** give the VM a permanent writable view of your real user profile.

---

## 7. Data Classification

| Data | Classification | Handling |
|---|---|---|
| Research drafts | Confidential | Never commit to repo. Sanitize if used in evidence. |
| Portal software | Internal | Can be referenced in docs. |
| Lab charter | Internal | Can be committed. |
| Lab logs and evidence | Internal | Sanitize before commit. Remove real IPs, usernames, credentials. |
| Snapshots | Internal | Keep local. Do not commit VM images. |

**Note:** Classification without handling rules is theater. These handling rules are minimal but real: sanitize before commit, never commit credentials, never commit real personal data.

---

## 8. RAM Allocation

See `docs/lab-charter.md` Section 5 for the full hardware tier decision.

**Core Lab:**

| VM | RAM |
|---|---|
| kali-offsec | 2 GB |
| rivermark-portal | 2 GB |
| Host reserve | 6–8 GB |

**Extended Lab (sequential, never simultaneous):**

| VM | RAM |
|---|---|
| rivermark-ws01 | 4 GB |
| rivermark-siem | 6 GB |

**Rule:** `rivermark-ws01` and `rivermark-siem` never run at the same time on this host. Power on only what the current mission needs. Snapshot the rest.

---

## 9. Boot and Addressing Notes

**Internal Network has no DHCP server.** VirtualBox does not provide IP addresses on Internal Network. You must configure static IPs manually:

- Ubuntu Server: netplan (`/etc/netplan/*.yaml`)
- Kali: NetworkManager or `/etc/network/interfaces`
- Windows: Network adapter settings

**The host cannot ping the lab on Internal Network.** This is intentional. If you need host-to-lab access, add a temporary Host-only adapter, then remove it.

**Default route behavior with dual NICs:**

- Traffic to `1.2.3.4` uses the NAT interface (default route).
- Traffic to `10.20.30.20` uses the Internal interface (directly connected network).
- This is why Kali never gets NAT.

---

## 10. Diagram Maintenance

Update this document when:

- A new VM is added.
- An IP address changes.
- A network mode changes.
- A new network segment is added.

Keep diagrams in ASCII or draw.io. Commit changes with a meaningful message.