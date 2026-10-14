# Baseline — What Normal Looks Like on rivermark-portal

**Date:** 2026-10-09
**Author:** Junior Security Analyst
**Purpose:** This is the reference for "expected" activity on the portal. Anything that contradicts this page is either a finding, a misconfiguration, or an incident. Without this page, detection is guesswork.

---

## 1. Who may SSH, from where, as which account

| Account | May SSH? | Allowed source plane | Source IP expected | sudo? |
|---|---|---|---|---|
| sysadmin | Yes | Management plane only | 192.168.56.1 (Windows host) | Yes |
| dev_amine | Yes | Lab LAN only | 10.20.30.10 (Kali) | No |
| dev_sarah | Yes | Lab LAN only | 10.20.30.10 (Kali) | No |
| contractor_old | **No** | — | — | — |

**Additional normal activity:**

- `sysadmin` may also SSH from the portal's own console (physical / VirtualBox window).
- No account may SSH from the public internet.
- Root login is not permitted.

---

## 2. What a normal dev_amine session looks like

A normal developer session, from Kali (10.20.30.10):

1. `ssh dev_amine@10.20.30.20`
2. Correct password entered
3. Server logs: `Accepted password for dev_amine from 10.20.30.10`
4. Server logs: `pam_unix(sshd:session): session opened for user dev_amine`
5. Developer runs `id` → sees uid=1002(dev_amine), groups dev_amine and users
6. Developer runs `sudo something` → **denied** (dev_amine is not in sudoers)
7. Server logs (in the sudo unit, not sshd): `sudo: dev_amine : user NOT in sudoers`
8. Developer runs `exit`
9. Server logs: `session closed for user dev_amine`

**That is the full shape of a normal developer interaction.** Any deviation is worth noting:
- A successful `sudo` for dev_amine would be abnormal (privilege escalation or config drift).
- Multiple failed passwords in a row from a single source would be abnormal (brute-force).
- A login from a different source IP would be abnormal (attacker using the developer's account).

---

## 3. What a normal sysadmin session looks like

A normal operator session, from Windows (192.168.56.1):

1. `ssh sysadmin@192.168.56.20`
2. Correct password entered
3. Server logs: `Accepted password for sysadmin from 192.168.56.1`
4. `sudo id` → succeeds, becomes root
5. Server logs (sudo unit): `sudo: sysadmin : TTY=pts/0 ; USER=root ; COMMAND=/usr/bin/id`
6. `exit`

**Additional normal activity for sysadmin:**
- Reading logs (`sudo journalctl ...`)
- Reading configuration files (`sudo cat /etc/netplan/...`)
- Editing netplan, editing sshd_config, restarting services
- SSHing from Windows PowerShell over the management plane

**Anything sysadmin does is normal **only if it comes from 192.168.56.1**. A sysadmin login from `10.20.30.10` (Kali) would be abnormal — Kali is not on the management plane, and sysadmin is not supposed to connect from the lab LAN.

---

## 4. What exists that is NOT normal in production, but is accepted in this lab

These are findings. They are intentional today so later missions can test detection. They are **not** fixes for Mission 3.

| Finding | Why it is not production-ready | Lab decision |
|---|---|---|
| **Password authentication is enabled** | Production best practice is SSH keys only | Kept on. Later missions will detect brute-force. |
| **`contractor_old` exists in `/etc/passwd` (locked, nologin, no sudo)** | Not a defect — this is a detection target. Deleting the account removes the signal. | Kept deliberately. Any authentication attempt is a finding. |
| **sshd listens on all interfaces (`0.0.0.0:22`)** | Production would bind to specific interfaces or use a bastion | Kept open on both planes. This is the finding. |
| **No central logging / no SIEM** | Production sends logs off-host to a SIEM | Local journald + rsyslog only. Wazuh comes in Extended Lab. |
| **No firewall (`ufw` inactive)** | Production would filter by port and source | Kept open. Later missions test what this exposes. |
| **`sysadmin` is the only sudo account** | Some environments require separate break-glass accounts | Accepted for now. |

---

## 5. Clock and time

- Both portal and Kali report UTC.
- Within a few seconds of each other. No measurable skew.
- The earlier 20-second difference was typing latency between terminals, not clock drift.

**Implication for detection:** time-correlation between portal logs and Kali activity is reliable. Later, when Wazuh ships logs to a central manager, ingestion time may differ from event time. When that happens, use the event timestamp from the message body, not the SIEM ingest timestamp.

---

## 6. Source IP is not an allow-list in Core Lab

**Important limitation:** In Core Lab, the developer laptop **and** the attacker both use `10.20.30.10` — because Kali is standing in for both roles. Kali is the attacker **and** the simulated developer machine.

**Consequence:**

- A rule like "alert if SSH is not from 10.20.30.10" would be useless.
- A rule like "alert if SSH is from 10.20.30.10" would fire on every legitimate developer login.
- **Source IP cannot be the primary signal in Core Lab.**

Detections in this lab must rely on:
- **Account** — which username authenticated
- **Result** — accepted vs failed
- **Time** — when, and how frequently
- **Behavior** — what happened after login (sudo denial, unusual command, etc.)

In Extended Lab, `rivermark-ws01` will be added at `10.20.30.30` as the true analyst workstation. At that point, `10.20.30.10` remains dirty (attacker) and `.30` is clean. Source-IP logic becomes meaningful.

---

## 7. What normal does NOT look like

These are examples of activity that would contradict this baseline:

- SSH login as `sysadmin` from `10.20.30.10` — the management plane is the only normal sysadmin path.
- SSH login as `dev_amine` from `192.168.56.1` — developers are not supposed to use the management plane.
- SSH login attempt as `contractor_old` — the account is locked; any attempt is noteworthy.
- Multiple failed passwords in a short window from a single source — brute-force pattern.
- Successful `sudo` for `dev_amine` or `dev_sarah` — privilege boundary broken.
- SSH login as `root` with a password — should be impossible (permitrootlogin without-password).
- Login from any source outside `10.20.30.0/24` or `192.168.56.0/24` — no other path exists.
- Session that runs unusual commands (shells spawned by web processes, curl to external IPs, base64-encoded strings).

---

## 8. Two log sources

Baseline note for later detection work:

| Source | Where | Content |
|---|---|---|
| journald | `journalctl` | sshd, sudo, systemd-logind, PAM events; structured fields |
| `/var/log/auth.log` | rsyslog | Same events in plain text; needs `grep -a` on this host |

Detection can be built on either. journald is preferable because fields like `_SYSTEMD_UNIT`, `SYSLOG_IDENTIFIER`, and `_UID` are structured and queryable. Plain text requires regex parsing.

---

## 9. Summary

**Normal:** dev_amine and dev_sarah connect from `10.20.30.10` to `10.20.30.20` with passwords, run read-only commands, get denied on sudo, log out. sysadmin connects from `192.168.56.1` to `192.168.56.20` with a password, uses sudo to read logs, log out.

**Not normal:** any login attempt as `contractor_old`, any successful sudo by a developer, any root login, any sysadmin login from the lab LAN, any burst of failed passwords, any connection from outside the two known subnets.

**Not production-ready yet:** password auth, contractor_old still in passwd, sshd on all interfaces, no SIEM, no firewall.

---

**End of baseline.**


---

## 10. HTTP portal — normal activity

### Who uses the portal

- `analyst_yassine` — the web user. Logs in from the lab LAN (Kali at 10.20.30.10 in Core Lab).
- No one else has web credentials in this lab.
- The portal is reachable only on port 80 (nginx). The app itself is on `127.0.0.1:8080` and is not exposed.

### What a normal session looks like

1. `GET /login` → returns the login form (200)
2. `POST /login` with correct password → 302 redirect to `/drafts`
3. `GET /drafts` with valid cookie → HTML list of three fictional drafts (200)
4. `GET /drafts/draft-001-acme.txt` with valid cookie → the draft contents (200)
5. `POST /logout` → 302 redirect to `/login`

### What is not normal

- `POST /login` returning 200 repeatedly from the same source in a short window — brute-force pattern
- `POST /login` with a username other than `analyst_yassine`
- `GET /drafts` without a cookie (302) many times — probing
- `GET /admin`, `GET /wp-login`, or other paths that do not exist — scanning
- Requests from IPs other than `10.20.30.10`
- The app binding to `0.0.0.0` instead of `127.0.0.1` (would mean exposure)
- Any successful login followed by unusual requests (unusual paths, unusual user-agent)

### What is not production-ready (accepted in the lab)

- HTTP only — no TLS
- Password stored in plaintext at `/etc/rivermark/secret`
- Sessions in memory, lost on restart, no expiry
- No CSRF protection
- No rate limiting
- App journal logs only login attempts, not reads
- nginx access log does not log POST bodies — username is not visible

### Detection implication — two logs, two scopes

| Question | nginx log | App journal |
|---|---|---|
| Every request | Yes | No |
| HTTP status | Yes | No |
| Username in login | **No** | Yes |
| Login result | Only via status | Yes, explicit |
| Draft reads | Yes | No |

**To detect a login failure, watch the app journal.**
**To detect post-login activity, watch the nginx log.**
**Neither log alone is sufficient.**

### Clock

The portal clock is currently **4 days behind** real time and not synchronized (no route to NTP). Incident reports in the lab must record both the portal time and the real UTC time to avoid confusion. This will be fixed in hardening.

### nginx alone cannot answer "did Yassine log in?"

nginx can see `POST /login 200` (failure) vs `POST /login 302` (success) — so it can tell **a login succeeded or failed**. But it cannot tell **who** logged in, because the username is in the POST body and nginx does not log request bodies by default.

**The app journal is required to answer who.**