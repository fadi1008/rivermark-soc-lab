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





## 2026-10-09 — Mission 3 Task A: Management plane + shared folder finding

### Management plane documented

**Decision:** The Host-only adapter on the portal (`enp0s9`, `192.168.56.20/24`) is a management plane, not a lab LAN. It is used only for operator access from the Windows host. Kali is never on `192.168.56.0/24`.

**Rules:**
- Only the portal has the Host-only NIC.
- Kali is on `rivermark-lan` only.
- No default route via `192.168.56.1` on either side.
- Shared folders are off on both VMs.
- Attack tools never run from the Windows host. The host is the operator, not the adversary.
- SSH from Windows uses `192.168.56.20` → `sysadmin` only.
- SSH from Kali uses `10.20.30.20` → developers only.

**Verification:**
- Portal `ip -br a`: `enp0s3 10.20.30.20/24`, `enp0s9 192.168.56.20/24`
- Portal `ip route`: no default route
- Kali `ip -br a`: `eth0 10.20.30.10/24`, `eth1` DOWN
- Kali `ip a | grep 192.168.56`: empty output — confirmed Kali is not on the management plane
- Kali `ip route`: no default route

### Shared folder finding (failed control)

**Finding:** Both VMs had VirtualBox shared folders enabled:
- `hadoop` → `C:\Users\fadib\Desktop\hadoop` (inside real user profile)
- `mediasf_win` → `C:\mediasf_win` (at C: root)

**Risk:** The lab VMs had write access to my real Windows user profile. This violates the charter's trust boundary rule: "the lab VM should not get a permanent, writable view of your real user profile."

**Action:**
- Removed both shared folders from VirtualBox settings on both VMs.
- Unmounted leftover `vboxsf` mounts inside each VM.
- Removed empty `/media/sf_*` directories.
- Rebooted both VMs.

**Verification after reboot:**

Portal:
- `ls /media/` → empty
- `mount | grep -i vbox` → empty

Kali:
- `ls /media/` → only `cdrom`, `cdrom0`
- `mount | grep -i vbox` → empty
- `lsmod | grep vbox` → `vboxguest` still loaded (Guest Additions fine, shared folders gone)

**Lesson:** VirtualBox shared folders cross the trust boundary the charter defines. Guest Additions alone (clipboard, drag-and-drop) is acceptable. Shared folders into the real user profile are not.

### Senior review answers

1. **Kali adapter list:** `eth0` = Internal (`rivermark-lan`), `eth1` = NAT (cable disconnected). No Host-only NIC. Shared folders: removed on 2026-10-09.
2. **Wait-online change:** systemd drop-in override at `/etc/systemd/system/systemd-networkd-wait-online.service.d/override.conf` with `ExecStart=/bin/true`. Result: boot time 3 min → 13 s.
3. **Clock skew direction:** No measurable skew. Both VMs show the same UTC time within a few seconds. The earlier 20-second difference was typing latency between terminals, not clock drift.
4. **Windows adapter 172.16.48.1:** No longer present. Current home gateway is `172.16.20.1` on adapter `Wi-Fi 2` (`172.16.20.238/24`). The earlier `172.16.48.1` was a previous network's gateway. `Ethernet 2` is the VirtualBox Host-only adapter (`192.168.56.1/24`) with no gateway — correct and intentional.
5. **ISO hash in decisions.md:** yes — Ubuntu 24.04.5, SHA256 verified against Canonical on 2026-09-25.




## 2026-10-09 — Mission 3 Task B: Account creation and privilege boundary

### Accounts created on rivermark-portal

| Account | UID | Shell | sudo? | passwd -S |
|---|---|---|---|---|
| sysadmin | 1001 | /bin/bash | yes — (ALL : ALL) ALL | P |
| dev_amine | 1002 | /bin/bash | no | P |
| dev_sarah | 1003 | /bin/bash | no | P |
| contractor_old | 1004 | /usr/sbin/nologin | no | L |

All accounts persist after reboot.

### Privilege boundary

- `sysadmin` is the only account with sudo.
- `dev_amine` and `dev_sarah` cannot run sudo.
- `contractor_old` cannot authenticate (locked) and cannot get a shell (nologin).
- Verified with `sudo -l -U <user>` — the effective policy check, not group membership alone.

### Method for contractor_old

- `adduser --disabled-password --gecos "Former Contractor" contractor_old`
- `usermod -s /usr/sbin/nologin contractor_old`
- `passwd -l contractor_old`

Two independent controls: locked password + nologin shell. Locking alone still allows login if the password is reset; nologin alone still permits authentication attempts. Doing both means the account cannot be used at all.

### Why keep contractor_old instead of deleting it

Deleting the account removes it from `/etc/passwd` — it becomes invisible. Any later authentication attempt would fail silently with no user record to correlate. Keeping it disabled preserves the identity so that any attempt to use it is **visible in the logs** and can trigger a detection.

In production, we would keep it for the same reason: "this user should not have come back" is a useful alert, and deleted accounts cannot produce it.

### Verification commands

- `id <user>` — confirm UID, GID, supplementary groups
- `sudo -l -U <user>` — confirm effective sudo policy
- `sudo passwd -S <user>` — confirm password state (P = set, L = locked, NP = no password)
- `ssh contractor_old@10.20.30.20` from Kali — `Permission denied`
- `sudo su - contractor_old` on the portal — `This account is currently not available.`

### Investigation answers

**1. `adduser` vs `useradd`:**
`useradd` is the low-level binary. It creates the account but does not create a home directory by default, does not prompt for a password, and does not copy `/etc/skel`. `adduser` is a friendly wrapper around `useradd` — it creates the home directory, prompts for a password, copies skeleton files, and adds the user to standard groups.

**2. `passwd -l`, `usermod -L`, `usermod -s /usr/sbin/nologin`:**
- `passwd -l` prepends `!` to the password hash in `/etc/shadow`. The user cannot authenticate.
- `usermod -L` does effectively the same (locks the password field).
- `usermod -s /usr/sbin/nologin` changes the login shell. Even if authentication succeeded, the shell immediately exits. No interactive session.

They are different controls and combine for stronger protection.

**3. `/etc/sudoers.d/` vs editing `/etc/sudoers`:**
`/etc/sudoers.d/` is a drop-in directory. Each rule is a small file. Advantages: (a) a syntax error breaks only that file, not sudo globally; (b) package upgrades that replace `/etc/sudoers` do not overwrite drop-ins; (c) easier to audit and remove.

**4. `sudo -l -U dev_amine`:**
It asks sudo "what is dev_amine allowed to run?" without logging in as that user. It is the correct verification because it shows the **effective** policy. A user could be in the `sudo` group with no rules, or not in the group but with rules via a drop-in. `sudo -l -U` is the ground truth.

**5. `passwd -S dev_amine`:**
Output format: `username status last_change min max warn inactive expire`.
Second field is status: `P` = password set, `L` = locked, `NP` = no password. This is the field detection rules will use to distinguish locked vs active accounts.

### Lesson

An account left behind is an identity an attacker does not have to create. Disabled is not deleted — disabled is detectable.





## 2026-10-09 — Mission 3 Task C: SSH paths proven

### SSH listener

`ss -tlnp | grep :22` shows sshd listening on `0.0.0.0:22` and `[::]:22` — all interfaces. The portal is reachable on both the lab LAN (`enp0s3`, 10.20.30.20) and the management plane (`enp0s9`, 192.168.56.20).

### SSH daemon configuration

`sudo sshd -T | grep -Ei 'permitroot|passwordauthentication|pubkeyauthentication|port '`:

- `port 22`
- `permitrootlogin without-password` — root cannot log in with a password
- `pubkeyauthentication yes`
- `passwordauthentication yes` — the company still uses passwords. Finding for later, not a bug today.

### Firewall

`ufw` is inactive. No filtering. Finding to revisit in the hardening phase.

### Path 1 — Kali → portal (lab LAN)

- Command: `ssh dev_amine@10.20.30.20` from Kali
- Result: success
- Session: `dev_amine@rivermark-portal`
- Source IP the portal sees: `10.20.30.10`
- `id` → uid=1002, groups dev_amine + users
- `sudo -l` → "user dev_amine may not run sudo on rivermark-portal"

### Path 2 — Windows → portal (management plane)

- Command: `ssh sysadmin@192.168.56.20` from Windows PowerShell
- Result: success
- Session: `sysadmin@rivermark-portal`
- Source IP the portal sees: `192.168.56.1`
- `sudo id` → `uid=0(root) gid=0(root)`

### Control — Kali → management plane

- `ssh dev_amine@192.168.56.20` → `Network is unreachable`
- `ping -c 2 192.168.56.20` → `Network is unreachable`
- Confirms Kali has no route to `192.168.56.0/24`. Management plane is portal-only.

### Concurrent sessions observed

While both paths were active:

`who` showed:

    sysadmin   pts/0  2026-10-09 14:11 (192.168.56.1)
    dev_amine  pts/1  2026-10-09 14:12 (10.20.30.10)

`ss -tnp | grep :22` showed:

    ESTAB  10.20.30.20:22    10.20.30.10:51752    ← Kali (lab LAN)
    ESTAB  192.168.56.20:22  192.168.56.1:61102   ← Windows (management plane)

Two planes are active at the same time. The server records the source IP of the interface the connection arrived on. This is the basis for later source-vs-plane detection.

### Deeper — what would break if sshd were bound to 10.20.30.20 only

If `ListenAddress 10.20.30.20` were set in sshd_config:

- Windows could no longer SSH to `192.168.56.20`. The management plane would break.
- The only path into the portal would be from the lab LAN, which is the attacker's network.
- Loss-of-management-plane scenario: if the lab LAN is flooded or blocked, there would be no out-of-band path.

Binding to a single IP is a valid production control, but it must match the operational reality. In this lab, the portal must be reachable on both planes.



## 2026-10-09 — Still-open answers (from Mission 2 review)

1. **Wait-online change:** systemd drop-in override at
   `/etc/systemd/system/systemd-networkd-wait-online.service.d/override.conf`
   with: 
[Service]
ExecStart=
ExecStart=/bin/true
Result: boot time 3 min → 13 s. No netplan `optional: true` set.

2. **Which clock is ahead:** Neither. Portal and Kali show the same UTC time
within seconds. The earlier 20-second difference was typing latency.

3. **Which Windows adapter owns the home gateway:** Wi-Fi 2
(172.16.20.238/24) with gateway 172.16.20.1. The earlier 172.16.48.1
was a previous Wi-Fi network's gateway; no longer present.
Ethernet 2 is the VirtualBox Host-only adapter (192.168.56.1/24),
no gateway.

4. **ISO hash match:** Yes. Ubuntu 24.04.5 Server,
sha256 = 97f3d7ffb032c3eb3b23d2c8be9cc76e60c2c1f2c0146ba5ba9fe01cafae0fd8,
verified against Canonical on 2026-09-25.