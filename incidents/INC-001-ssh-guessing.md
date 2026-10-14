
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
