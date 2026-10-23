# SSH Detection Rules

**Date:** 2026-10-16
**Target:** rivermark-portal (10.20.30.20)

## Rule 1 — SSH_GUESS
**Intent:** Detect a brute-force/spray attack.
**Threshold:** 5 failed attempts within 60 seconds from the same source IP.
**Why 5:**
- Normal baseline (developer typo): 1 failure.
- INC-001 (attack): 19 failures in 16 seconds.
- 5 is high enough to ignore a typo, but low enough to trigger immediately during an attack.

## Rule 2 — SSH_SUCCESS_AFTER_FAILURES
**Intent:** Detect a successful compromise after a burst of failures.
**Threshold:** 1 `Accepted password` after >= 5 prior `Failed password` attempts from the same IP within 60 seconds.
**Critical Detail:** The window does NOT reset on success. INC-001 showed 2 failures happening *after* the successful login because Hydra was running parallel tasks. The rule must catch the success even if the attack hasn't finished.

## Rule 3 — SSH_UNEXPECTED_ACCOUNT
**Intent:** Detect enumeration and attempts on non-standard accounts.
**Threshold:** 1 event for any `Invalid user`, `root`, `admin`, or `contractor_old`.
**Why:** Baseline says developers only log in as `dev_amine` or `dev_sarah`. Attempting `root` is always malicious.

---

## Alert Text — SSH_SUCCESS_AFTER_FAILURES

> **HIGH — SSH_SUCCESS_AFTER_FAILURES**
> Successful SSH login for `<user>` from `<src>` at `<timestamp>` UTC, after `<N>` failed attempts from the same source IP within 60 seconds.
> 
> **Next check:** Look at the same source IP in the nginx access log (`/var/log/nginx/access.log`) and the app journal (`journalctl -u rivermark-portal.service`) for the same 1-minute window. Did this session issue any `GET /drafts` request? Did it run any `sudo` command?

---

## Misses — What these rules do NOT cover
1. **Crown Jewel Access:** SSH rules cannot prove a user read a file. Only nginx logs can show `GET /drafts`.
2. **Web Login Sprays:** A brute-force against the web portal (`POST /login`) will not trigger these SSH rules. It requires parsing the nginx/app logs.
3. **Ubiquitous "Invalid user":** If Ubuntu changes the log wording, Rule 3 might fail. We mitigate this by also explicitly checking for `root` and `admin`.


## Detection Updates Post-Hardening (Task B)
Password authentication is now disabled. SSH logs will no longer show `Failed password` or `Accepted password`. They will show `Failed publickey` and `Accepted publickey`. 
- `SSH_GUESS` and `SSH_SUCCESS_AFTER_FAILURES` regexes in `detect_ssh.py` must be updated to match `Failed publickey` and `Accepted publickey` to remain functional.
- Residual threat: A stolen private key is now the attack vector, not a wordlist.



### Change in Rule Meaning (Mission 10)
`SSH_GUESS` no longer means "password spray." Since passwords are disabled, it now means "several sessions died in preauth." A missing key, a client that gives up, and Hydra all end in `Connection closed by authenticating user [preauth]`. The threshold of 5 in 60 seconds keeps a single mistake from paging us.