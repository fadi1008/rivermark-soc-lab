# Incident Response Plan (INC-001 & INC-002)

## Pre-Response Preservation
- Snapshot taken: `m8-pre-response`
- Logs for INC-001 (SSH) and INC-002 (Web) are preserved in the repo.
- Alert lines captured (see bottom).

## Containment Plan
1. **Change passwords:** ONLY `dev_amine` (SSH) and `analyst_yassine` (Web).
2. **Kill sessions:** Restart the Python app (kills all web sessions). Kill any active SSH sessions for dev_amine.
3. **Do NOT block:** 10.20.30.10 will remain unblocked. The business (developers/analysts) uses this IP.
4. **Do NOT rotate:** `sysadmin` or `dev_sarah` (they were not compromised).
5. **Verification:** Old credentials must fail. New credentials must work. Portal must still function.

## Appendix — Alert Lines
*SSH Alerts (INC-001):*
[MEDIUM] SSH_GUESS @ 2026-10-15 10:01:14 — 19 failed attempts from 10.20.30.10
[HIGH] SSH_SUCCESS_AFTER_FAILURES @ 2026-10-15 10:01:46 — Login for dev_amine from 10.20.30.10 after 16 failures
[MEDIUM] SSH_UNEXPECTED_ACCOUNT @ 2026-10-15 10:01:14 — unexpected account root from 10.20.30.10

*Web Alerts (INC-002):*
[MEDIUM] WEB_LOGIN_GUESS @ 2026-10-15 10:08:07 — 4 failed logins from 10.20.30.10
[HIGH] WEB_DRAFT_AFTER_GUESS @ 2026-10-15 10:08:10 — User analyst_yassine read /drafts/draft-001-acme.txt from 10.20.30.10 after a login burst


## Execution & Verification

**Persistence Check Results (Task B):**
- Active sessions: [Paste "None found" or the output]
- Authorized keys: None found for dev_amine or root.
- Cron jobs: None found.
- Suspicious /tmp files: None found.
- **Conclusion:** No persistence established. The attacker only used the credentials, no backdoors were planted.

**What was changed (Task C):**
- `dev_amine` Linux password rotated.
- `analyst_yassine` web password rotated in `/etc/rivermark/secret`.
- Web sessions invalidated via `systemctl restart rivermark-portal`.

**What was NOT changed:**
- `sysadmin` and `dev_sarah` passwords left intact.
- IP 10.20.30.10 was NOT blocked (business continues to use this IP).
- SSH password authentication remains enabled.

**Verification:**
- Old SSH password rejected.
- Old Web password rejected (HTTP 200).
- New Web password accepted (HTTP 302), new session successfully read the draft.
- New SSH password accepted, but `sudo` and `cat /srv/rivermark/drafts/` both denied.

**Residual Risk (Plain Language):**
The stolen passwords are dead, and the specific sessions are closed. However, the attacker knows `dev_amine` and `analyst_yassine` are valid accounts. Because our detection threshold for failed logins is 3, a single quiet retry with the old password will look like a normal typo to our alerting system. We have no mechanism yet to ban a repeat offender.

## Appendix — Alert Lines (Missions 6 & 7)

*SSH Alerts (INC-001):*
[MEDIUM] SSH_GUESS @ 2026-10-15 10:01:14 — 19 failed attempts from 10.20.30.10
[HIGH] SSH_SUCCESS_AFTER_FAILURES @ 2026-10-15 10:01:46 — Login for dev_amine from 10.20.30.10 after 16 failures
[MEDIUM] SSH_UNEXPECTED_ACCOUNT @ 2026-10-15 10:01:14 — unexpected account root from 10.20.30.10

*Web Alerts (INC-002):*
[MEDIUM] WEB_LOGIN_GUESS @ 2026-10-15 10:08:07 — 4 failed logins from 10.20.30.10
[HIGH] WEB_DRAFT_AFTER_GUESS @ 2026-10-15 10:08:10 — User analyst_yassine read /drafts/draft-001-acme.txt from 10.20.30.10 after a login burst