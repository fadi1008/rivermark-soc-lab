
# INC-001 — SSH Password Guessing Against rivermark-portal

**Status:** In progress — hypothesis written BEFORE the attack  
**Date:** 2026-10-14  
**Analyst:** Junior Security Analyst

---

## Hypothesis (written before running Hydra)

Predictions of what the sshd journal will show after a small SSH guessing attack.

### 1. Real account, wrong password (e.g., dev_amine with wrong password)

**What I expect:**

Failed password for dev_amine from 10.20.30.10 port XXXXX ssh2


### 2. Username that does not exist (e.g., no_such_user)

**What I expect:**


Invalid user no_such_user from 10.20.30.10 port XXXXX
...
Failed password for invalid user no_such_user from 10.20.30.10 port XXXXX ssh2


The literal substring `invalid user` distinguishes this from a real-account failure.

### 3. root

**What I expect:** Either of two things; I am not sure yet:

- `Failed password for root from 10.20.30.10` — if sshd allows password authentication for root.
- `Invalid user root` — unlikely, because root exists.
- Some message tied to `PermitRootLogin prohibit-password` if the server refuses before checking the password.

Either result is data. I will capture what actually appears.

### 4. contractor_old (locked, nologin)

**What I expect:**


Failed password for contractor_old from 10.20.30.10 port XXXXX ssh2


The locked state is **not** visible in the sshd journal. From the log alone, this is indistinguishable from a wrong-password attempt against an active account.

### 5. Successful dev_amine login after failures

**What I expect:**


Accepted password for dev_amine from 10.20.30.10 port XXXXX ssh2
pam_unix(sshd:session): session opened for user dev_amine(uid=1002) by dev_amine(uid=0)


**Distinguishable from a normal dev_amine login?**

A single `Accepted password for dev_amine` line, in isolation, looks **identical** to a normal login. The only difference is context: a burst of approximately 30 failures immediately before it, from the same source IP.

**Conclusion:** The successful login is only suspicious in **context**, not in isolation. A rule that alerts on a single successful login would fire on normal traffic. A rule that counts failures in a time window, then watches for a success, would fire on this incident only.

---

## Attack plan (not yet executed)

- **Tool:** Hydra
- **Target:** `10.20.30.20` only (no other host, no web)
- **Usernames:** `root`, `admin`, `no_such_user`, `dev_amine`, `dev_sarah`, `contractor_old`
- **Passwords:** 5 entries, dev_amine's real password last
- **Stop condition:** dev_amine succeeds, or the list ends
- **Snapshot portal first:** `m5-pre-ssh-guess`
- **Packet capture:** Capture packets on the portal with `tcpdump` during the attack.





# INC-001 — SSH Password Guessing Against rivermark-portal

**Status:** Contained (no action taken)  
**Detected:** 2026-10-14 15:58:30 UTC  
**Analyst:** Junior Security Analyst  
**Severity:** Medium

---

## Summary

Between 15:58:30 and 15:58:46 UTC on 2026-10-14, a 16-second burst of SSH authentication attempts originated from 10.20.30.10 (Kali, lab LAN) against rivermark-portal (10.20.30.20). The attack tried a small list of usernames and passwords. One credential was correct: dev_amine. The successful session was short (login, exit) and did not read any draft.

---

## Timeline (portal UTC)

| Time (UTC) | Event |
|---|---|
| 15:58:30 | Attack starts — first root attempt disconnected |
| 15:58:31 | 4× `Failed password for root` (parallel) |
| 15:58:32 | 3× `Failed password for root` |
| 15:58:35 | Retry on port 34924 — second attempt on same connection |
| 15:58:37 | Final root disconnect + PAM failure summary |
| 15:58:40 | `Invalid user admin`, `Invalid user no_such_user` |
| 15:58:41–15:58:43 | Multiple `Failed password for dev_amine` and `Failed password for invalid user no_such_user` |
| **15:58:44** | **`Accepted password for dev_amine from 10.20.30.10 port 58990`** |
| 15:58:44–15:58:45 | 2 more `Failed password for dev_amine` from parallel tasks still running |
| 15:58:46 | Last `Connection closed by dev_amine [preauth]` |

---

## Source IP

10.20.30.10 — Kali, lab LAN. Note: this is also the address dev_amine normally logs in from. Source IP alone is not the signal.

---

## Counts (corrected)

| Metric | Value |
|---|---|
| Password attempts (Failed + Accepted) | 20 |
| TCP connections (unique source ports) | 16 |
| Invalid user enumeration lines | 6 |

**Why 30 SYNs but 16 connections:** ~14 SYNs were TCP retransmissions.
Counting unique source ports gives the true connection count.

**Why 16 connections but 20 attempts:** some connections delivered more
than one password attempt before sshd closed them (MaxAuthTries).
The pcap shows port 34924 carrying two root attempts.

**Why the journal has 26 raw lines but only 20 attempts:** each attempt
generates 2–5 journal lines (PAM check, PAM failure, Failed password,
Connection closed). Counting raw lines over-counts.

**For detection thresholds, use the 20 attempts figure, not the raw
journal lines.** The rule counts events, not log lines.

## The success

**Line I would keep:**

    Oct 14 15:58:44 rivermark-portal sshd[5317]: Accepted password for dev_amine from 10.20.30.10 port 58990 ssh2

**Why this is not a normal dev_amine session:**

A normal dev_amine login in the baseline is a single `Accepted password` line with no preceding burst. This success is preceded by 14 seconds of failed attempts from the same source IP, and followed by 2 more failed dev_amine attempts from parallel Hydra tasks.

**Key nuance:** the successful login did NOT stop the failures. Hydra with `-t 4` runs 4 parallel tasks; when one found the credential, the other 3 were mid-attempt. Detection rules must not assume that success ends the burst.

---

## Invalid user example

    Oct 14 15:58:40 rivermark-portal sshd[5316]: Invalid user no_such_user from 10.20.30.10 port 58986

`Invalid user` means the username does not exist. `Failed password for <name>` (without `invalid user`) means the username exists.

---

## contractor_old

Not attempted in this attack. Hydra was run with `-f` (stop on first success). dev_amine succeeded before the parallel tasks reached contractor_old. The list was not exhausted.

**Finding:** `-f` changes the shape of an incident. The usernames tried depend on where in the list the successful credential sits, not on the attacker's full intent.

---

## root

    Oct 14 15:58:32 rivermark-portal sshd[5299]: Failed password for root from 10.20.30.10 port 34924 ssh2

`root` behaves like any other real user. The `permitrootlogin without-password` setting affects successful authentication (blocks password-only root login) but does not change how failed password attempts are logged.

---

## dev_sarah

Not observed in the attack window. Hydra did not reach her in the list before `-f` stopped the run.

---

## Impact

The successful dev_amine session could:

- Run user-level commands as dev_amine.
- See files owned by dev_amine.
- NOT read drafts (`/srv/rivermark/drafts/` is `750 rivermark:rivermark`).
- NOT sudo (verified in Mission 3).
- NOT access `/srv/rivermark/` (permission denied).

**What the attacker actually got:** a valid shell as dev_amine. No privilege escalation. No access to the crown jewel. The account is real, but its file permissions limit what it can reach.

**"Compromised" is not a binary.** This session is a real account on the right port from the developer's usual address. It is not the crown jewel unless files moved. They did not.

---

## MITRE ATT&CK

- **T1110.001 — Password Guessing:** systematic attempt of a small username/password list. Fits.
- **T1078 — Valid Accounts:** the attacker used a valid developer credential to obtain a session. Fits, with the note: no post-compromise activity observed (no file access, no escalation, no lateral movement).

---

## What an alert should say (not implemented)

> More than 10 SSH authentication failures from a single source IP within 60 seconds, regardless of target account.

---

## What I did NOT do (yet)

- Did not disable the dev_amine account.
- Did not block the source IP.
- Did not restore the snapshot `m5-pre-ssh-guess`.
- Did not modify `sshd_config`.
- Did not modify the app.
- Did not enable a firewall.

The logs are the lab.

---

## Appendix — attacker command

    hydra -L /tmp/users.txt -P /tmp/passwords.txt -t 4 -f ssh://10.20.30.20

Attacker knowledge. The timeline does not depend on it.

---

## Appendix — packet capture

- **File:** `/tmp/ssh-attack.pcap`
- **Captured:** 416 packets
- **Filter:** `tcp port 22` on `enp0s3`
- **TCP SYN count:** 30
- **Unique source ports (true connection count):** 16


## Cross-log verification
nginx access log checked for 2026-10-14 15:58–15:59 UTC.
Result: No HTTP requests to `/drafts` from 10.20.30.10 during the SSH attack. 
Conclusion: The impact statement is confirmed. The attacker got a shell as `dev_amine`, but never touched the web portal or the draft files.