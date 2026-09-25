# Rivermark SOC Lab — Architecture

**Author:** fadi bellalah
**Date:** 24-9-2026
**Version:** 1.0

---

YOUR WINDOWS 11 HOST
(not on rivermark-lan)
|
VirtualBox
|
+---------------+---------------+
| |
[Internal Network] [Host-only adapter]
rivermark-lan (temporary, for evidence)
10.20.30.0/24 (dashed — not always cabled)
| |
+---------+---------+ |
| | |
[kali-offsec] [rivermark-portal] |
10.20.30.10 10.20.30.20 |
(adversary) (web + MySQL + SSH) |
| |
+-------------------------+
(SCP evidence out)


The Host-only adapter is drawn dashed because it is temporary, not permanently cabled. It is added only for evidence transfer, then disconnected.

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

| Mode | Internet? | Other VMs? | Host? | Home Wi-Fi devices? | Password spray can hit what is not a lab VM? |
|---|---|---|---|---|---|
| **Bridged** | Yes | Yes (same L2) | Yes | **Yes** | **Yes — home LAN and the internet** |
| **NAT** | Yes | No (unless port-forward) | Guest → host often works; host → guest does not without port-forward | No | **Yes — the internet (not the home LAN)** |
| **NAT Network** | Yes | Yes (same NAT network) | Limited (port-forward) | No | **Yes — the internet (not the home LAN)** |
| **Host-only** | No | Yes (same host-only network) | Yes (host has adapter) | No | **Yes — the host (your real Windows 11)** |
| **Internal Network** | No | Yes (same internal network) | **No** | No | **No, if you target lab IPs; other destinations fail closed (no route)** |

**Why Internal Network was chosen:**

- The host is not on the lab LAN. This prevents the host from accidentally interacting with attack traffic.
- No internet by default. A mistyped scan cannot leave the lab.
- Isolation is enforced at the hypervisor level.

**Why Bridged was rejected:**

- Bridged places Kali on the same Layer 2 network as home devices (router, printer, phones, other PCs).
- A password spray against `192.168.1.0/24` from Kali would target real devices.
- This is unacceptable and potentially illegal.

**Why NAT on Kali is dangerous:**

- NAT hides you from the home LAN. It does not block outbound traffic.
- `hydra` against `ssh.example.com` from a NAT-attached Kali leaves your house and becomes a real attack.
- A mistyped target becomes a real attack because the default route points to the internet.

**Why Host-only is not harmless:**

- `192.168.56.1` is often the host. A spray there can lock your real account or hit real RDP/SSH on the host.
- Keep Host-only off Kali.

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

**Rule:** Only `rivermark-portal` ever gets a NAT adapter. Kali never gets NAT.

### Adapter layout on rivermark-portal

| Adapter | Network | Config | Default state |
|---|---|---|---|
| Adapter 1 | Internal `rivermark-lan` | Static `10.20.30.20/24`, no gateway | Always attached, cable connected |
| Adapter 2 | NAT | DHCP (used only during updates) | Present in VM config, cable disconnected by default |

### Update window procedure

1. Snapshot before.
2. In VirtualBox, tick **Cable Connected** on Adapter 2.
3. Boot. Confirm `ip route show default` shows NAT as default.
4. `sudo apt update && sudo apt upgrade -y`
5. Untick **Cable Connected** on Adapter 2.
6. Confirm `ip route show default` no longer shows a default route.
7. Snapshot after: `post-update-clean`.

**Why cable disconnect, not adapter delete:**

- VirtualBox keeps the NIC in the VM config when you disconnect the cable. Netplan still matches the same interface name (`ens3` stays `ens3`).
- No cable = interface down = no default route.
- Deleting or hot-adding the NIC renames the interface (`ens3` → `ens18`) and leaves the portal without `10.20.30.20` on next boot.

**Kali:** Adapter 1 Internal only. No Adapter 2. Ever.

**A missing default route is a security control.** After every change, `ip route show default` on the portal should show nothing — except during a documented update window. A surprise default route is an incident.

---

## 6. Evidence Transfer Rule

**Rule:** Use a temporary Host-only adapter for evidence transfer. Never use permanent shared folders.

**Why:** Internal Network means the host is not on `rivermark-lan`. This is a feature. But you need a way to move sanitized logs into the Git repo without bridged networking. Shared folders give the VM a permanent writable view of your real user profile — that is a trust risk.

**Procedure (Mission 2+):**

1. Power off `rivermark-portal`.
2. Add a Host-only adapter (second NIC).
3. Boot. The host can now reach the portal on a separate host-only subnet (e.g., `192.168.56.0/24`).
4. From the host, `scp` the sanitized log file out.
5. Power off. Disconnect the Host-only adapter (do not delete — same netplan reasoning).
6. Sanitize the file on the host (remove real IPs, usernames if needed).
7. Commit to Git.

**Note:** For Mission 2, you will not need SCP yet. You can paste evidence from the VM console. This procedure becomes relevant when evidence transfer becomes the bottleneck. Later, the realistic pattern is a persistent Host-only on the portal only, acting as a management plane. We will revisit then.

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
| rivermark-portal | 2 GB (can go to 2560–3072 MB if MySQL + nginx cause swapping) |
| Host reserve | 6–8 GB |

**Extended Lab (sequential, never simultaneous):**

| VM | RAM |
|---|---|
| rivermark-ws01 | 4 GB |
| rivermark-siem | 6 GB |

**Rule:** `rivermark-ws01` and `rivermark-siem` never run at the same time on this host. Power on only what the current mission needs. Snapshot the rest.

**Do not steal RAM from the host reserve** to make the portal larger. If the portal needs more, accept the swap or move to a larger host.

---

## 9. Boot and Addressing Notes

**Internal Network has no DHCP server.** VirtualBox does not provide IP addresses on Internal Network. You must configure static IPs manually:

- Ubuntu Server: netplan (`/etc/netplan/*.yaml`)
- Kali: NetworkManager or `/etc/network/interfaces`
- Windows: Network adapter settings

**The host cannot ping the lab on Internal Network.** This is intentional. If you need host-to-lab access, connect the temporary Host-only adapter, then disconnect it.

**Default route behavior with dual NICs:**

- Traffic to `1.2.3.4` uses the NAT interface (default route).
- Traffic to `10.20.30.20` uses the Internal interface (directly connected network).
- This is why Kali never gets NAT.

**After every adapter change, verify:**

```bash
ip addr          # confirm IPs
ip route         # confirm no surprise default route