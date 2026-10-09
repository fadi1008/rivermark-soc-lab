
# Baseline SSH Authentication Evidence

**Date:** 2026-10-09  
**Host:** rivermark-portal (Ubuntu 24.04.5, systemd 255)  
**Log sources:** `journalctl` (systemd journal) AND `/var/log/auth.log` (rsyslog present)

---

## Event 1 — Successful dev_amine login from Kali (lab LAN)

- UTC start time (from Kali): `2026-10-09 14:19:48`
- Source IP as I understand it: `10.20.30.10` (Kali, rivermark-lan)
- Source IP as the log states it: `10.20.30.10`
- One line I would keep if I could keep only one:

```text
Oct 09 14:19:58 rivermark-portal sshd[1698]: Accepted password for dev_amine from 10.20.30.10 port 41086 ssh2
```

- What the other lines add:

```text
Oct 09 14:19:58 sshd[1698]: pam_unix(sshd:session): session opened for user dev_amine(uid=1002) by dev_amine(uid=0)
Oct 09 14:19:58 systemd-logind[713]: New session 18 of user dev_amine.
Oct 09 14:19:58 (systemd): pam_unix(systemd-user:session): session opened for user dev_amine(uid=1002)
```

These prove a session existed and was tracked by systemd. They do NOT prove authentication succeeded — that is the "Accepted password" line. Counting all four would over-count one login as four events.

---

## Event 2 — Failed dev_amine login from Kali (lab LAN)

- UTC start time (from Kali): `2026-10-09 14:20:24`
- Source IP as I understand it: `10.20.30.10`
- Source IP in the log: `10.20.30.10`
- One line I would keep:

```text
Oct 09 14:21:09 rivermark-portal sshd[1872]: Failed password for dev_amine from 10.20.30.10 port 43966 ssh2
```

- What the other lines add:

```text
Oct 09 14:21:07 sshd[1872]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=10.20.30.10  user=dev_amine
Oct 09 14:21:15 sshd[1872]: Connection closed by authenticating user dev_amine 10.20.30.10 port 43966 [preauth]
```

The `pam_unix` line is the pre-auth failure notice. The `Failed password` line is the definitive authentication failure. The `Connection closed ... [preauth]` line appears when the client aborts; it adds no authentication information.

---

## Event 3 — Successful sysadmin login from Windows (management plane)

- UTC start time (from Windows): `2026-10-09 14:22:17`
- Source IP as I understand it: `192.168.56.1` (Windows host, Host-only management plane)
- Source IP in the log: `192.168.56.1`
- One line I would keep:

```text
Oct 09 14:22:27 rivermark-portal sshd[1895]: Accepted password for sysadmin from 192.168.56.1 port 51216 ssh2
```

- What the other lines add:

```text
Oct 09 14:22:27 sshd[1895]: pam_unix(sshd:session): session opened for user sysadmin(uid=1001) by sysadmin(uid=0)
Oct 09 14:22:27 systemd-logind[713]: New session 22 of user sysadmin.
Oct 09 14:22:44 sudo[1980]: sysadmin : TTY=pts/0 ; PWD=/home/sysadmin ; USER=root ; COMMAND=/usr/bin/id
Oct 09 14:22:44 sudo[1980]: pam_unix(sudo:session): session opened for user root(uid=0) by sysadmin(uid=1001)
Oct 09 14:22:44 sudo[1980]: pam_unix(sudo:session): session closed for user root
```

The first two lines confirm the SSH session started. The last three lines are a separate event: sysadmin successfully ran `sudo id` and became root. This is logged by the `sudo` service, not by sshd.

---

## Event 4 — dev_amine sudo denial (lab LAN)

- UTC start time (from Kali): `2026-10-09 14:23:14`
- Source IP as I understand it: `10.20.30.10`
- Source IP in the log: not present — this is a `sudo` event, not an sshd event
- The line I would keep:

```text
Oct 09 14:23:42 rivermark-portal sudo[2069]: dev_amine : user NOT in sudoers ; TTY=pts/0 ; PWD=/home/dev_amine ; USER=root ; COMMAND=/usr/bin/id
```

- What the other lines add: nothing. This is a single sudo event.

### FINDING — this event is invisible to an sshd-only rule

Running `journalctl -u ssh --since "15 minutes ago"` around 14:23 shows:

```text
14:23:27 Accepted password for dev_amine from 10.20.30.10 port 33246 ssh2
14:23:27 pam_unix(sshd:session): session opened for user dev_amine
...
14:24:13 pam_unix(sshd:auth): authentication failure; rhost=10.20.30.10 user=contractor_old
```

The sudo denial at **14:23:42** never appears in the sshd unit. It appears only in:

```bash
sudo journalctl --since "15 minutes ago" | grep -i sudo
```

**Lesson:** an sshd-only detection misses privilege misuse entirely.

---

## Event 5 — contractor_old login attempt from Kali (lab LAN)

- UTC start time (from Kali): `2026-10-09 14:24:10`
- Source IP as I understand it: `10.20.30.10`
- Source IP in the log: `10.20.30.10`
- One line I would keep:

```text
Oct 09 14:24:15 rivermark-portal sshd[2079]: Failed password for contractor_old from 10.20.30.10 port 34174 ssh2
```

- What the other lines add:

```text
Oct 09 14:24:13 sshd[2079]: pam_unix(sshd:auth): authentication failure; logname= uid=0 euid=0 tty=ssh ruser= rhost=10.20.30.10  user=contractor_old
Oct 09 14:24:19 sshd[2079]: Connection closed by authenticating user contractor_old 10.20.30.10 port 34174 [preauth]
```

### FINDING — locked account is not distinguished in the log

Even though `contractor_old` is locked in `/etc/shadow` (field = `L`), the sshd journal does NOT say "account is locked." It only says `Failed password`. From the log alone, this looks identical to a wrong-password attempt against a normal account.

**Consequence:** to detect "someone tried a disabled account," a rule must either:

- Compare the username against a known list of disabled accounts, OR
- Query `/etc/shadow` state, OR
- Use custom PAM logging that records the lock reason.

The plain-text `/var/log/auth.log` shows the same lines. The structured journal does not distinguish the lock reason.

---

## Summary — one login, many lines

A successful SSH login produces at least **four journal lines**: one for authentication (`Accepted password`), one for PAM session open (`pam_unix session opened`), one for systemd-logind (`New session`), and one for the user-level systemd instance. A failed login produces **two or three** (`pam_unix auth failure` + `Failed password` + optionally `Connection closed [preauth]`). A **sudo denial produces zero sshd lines** — it is logged only by the `sudo` service and is missed by any rule that watches only `journalctl -u ssh`.

The correct single line to alert on for each event:

| Event type | Single line |
|---|---|
| Successful auth | `Accepted password for <user> from <ip>` |
| Failed auth | `Failed password for <user> from <ip>` |
| Privilege misuse | `sudo: <user> : user NOT in sudoers` |

Counting all lines for one login over-counts by a factor of 3–4. Watching only sshd misses sudo entirely.

---

## Discovery checkpoint answers

**1. Which line proves the password was accepted?**

```text
Accepted password for <user> from <ip> port <p> ssh2
```

This is sshd's own authentication-success message, written after the password check passes.

**2. Which line only proves a session existed?**

```text
pam_unix(sshd:session): session opened for user <user>
```

This is written when PAM creates the session — it would appear even if the user disconnected one millisecond later.

**3. Which line would still be true if the user disconnected immediately?**

The `session opened` line. It records the start of the session, not its duration. The matching `session closed` line appears only when the session ends.

---

## Note on `/var/log/auth.log`

`grep` reported `binary file matches`. That happens when grep encounters a non-text byte in the file (rsyslog output can include control characters). To force text mode:

```bash
sudo grep -a dev_amine /var/log/auth.log | tail -20
```

Two log sources exist on this host:

| Source | Pros | Cons |
|---|---|---|
| `journalctl` | Structured fields (`_UID`, `_SYSTEMD_UNIT`, `SYSLOG_IDENTIFIER`), queryable, no parsing | Not a flat file; rules must use journald syntax |
| `/var/log/auth.log` | Plain text; simple regex rules work | No structured fields; `grep -a` needed on this host |

For later missions with Wazuh, both will likely be shipped. Note it now.

---

## After creating the file

```bash
git add evidence/baseline-ssh-auth.md
git commit -m "Mission 3 Task D: five SSH events captured, sudo-not-in-sshd finding documented"
git push
```


---

## Event 6 — Invalid user (nonexistent account) from Kali

- UTC time (from Kali): 2026-10-09 15:51:39
- Source IP: 10.20.30.10 (Kali, lab LAN)
- Full journal block:

```text
Oct 09 15:51:39 rivermark-portal sshd[2311]: Invalid user no_such_user from 10.20.30.10 port 43056
Oct 09 15:51:42 rivermark-portal sshd[2311]: pam_unix(sshd:auth): check pass; user unknown
Oct 09 15:51:42 rivermark-portal sshd[2311]: pam_unix(sshd:auth): authentication failure; rhost=10.20.30.10
Oct 09 15:51:44 rivermark-portal sshd[2311]: Failed password for invalid user no_such_user from 10.20.30.10 port 43056 ssh2
Oct 09 15:51:47 rivermark-portal sshd[2311]: Connection closed by invalid user no_such_user 10.20.30.10 port 43056 [preauth]
```

- The single line I would keep:

```text
Oct 09 15:51:44 rivermark-portal sshd[2311]: Failed password for invalid user no_such_user from 10.20.30.10 port 43056 ssh2
```

### Difference vs `Failed password for contractor_old`

- `Failed password for contractor_old` → the username exists in `/etc/passwd`; the password was wrong.
- `Failed password for invalid user no_such_user` → the username does not exist, so the login could never succeed, whatever password was typed.

The literal substring `invalid user` is the discriminator. It appears in three of the five lines (the first `Invalid user`, the `Failed password for invalid user`, and the `Connection closed by invalid user`).

### What an attacker learns from the difference

**User enumeration.** The attacker does not see these log lines. From the client side, both cases return the same `Permission denied`. But if the server leaks the difference some other way (for example, a different response time), the attacker can separate:
- Real accounts → worth password-guessing
- Nonexistent accounts → wasted attempts

On the defender side, the logs *do* show the difference. A defender who only watches for `Failed password for <name>` will miss enumeration entirely. Detection rules must match both patterns:
- `Failed password for <user>` — real account, wrong password
- `Failed password for invalid user <user>` — enumeration attempt

A burst of `invalid user` from one source is a stronger signal of scanning than a burst of failed passwords against one real account.

### Note

`pam_unix(sshd:auth): check pass; user unknown` is PAM's line for the same event. It appears only for nonexistent users. It is useful if a detection is built on PAM output rather than sshd output.