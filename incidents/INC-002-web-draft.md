# INC-002 — Web Login Brute Force and Draft Access

## Hypothesis (Written before execution)
1. **nginx logging:** A bad password returns HTTP 200. A good password returns HTTP 302. nginx will NOT contain the username.
2. **App logging:** The app journal will contain `login failure user=analyst_yassine` and `login success user=analyst_yassine`.
3. **SSH script:** `detect_ssh.py` will NOT fire on this attack because it only parses the SSH journal.
4. **Normal baseline:** A normal portal session is a single login (302), followed by GET /drafts (200). Our threshold must ignore this.



## Summary
On 2026-10-22, a brute-force attack was launched against the portal login form (`/login`) from 10.20.30.10. The attacker successfully guessed the password for `analyst_yassine` and used the session to read a confidential research draft.

## Timeline (Portal UTC)
- **14:45:40** — Hydra starts sending POST /login requests. 4 failures (200).
- **14:45:41** — Hydra submits the final password (`369258147`). Login succeeds (302).
- **14:45:41** — WEB_LOGIN_GUESS triggers based on the 4 failures.
- **14:45:41** — Attacker uses session cookie to request `/drafts/draft-001-acme.txt` (200).
- **14:45:41** — WEB_DRAFT_AFTER_GUESS triggers based on the draft read.

## Counts
*   Failed login attempts: 4
*   Successful logins: 1
*   Drafts read: 1 (`draft-001-acme.txt`)

## Impact
The confidential research file `draft-001-acme.txt` was read by an unauthorized party. SSH was not required; the attacker used valid credentials against the web portal.

## MITRE ATT&CK
*   **T1110.001 — Password Guessing:** The attacker guessed the web password.
*   **T1078 — Valid Accounts:** The attacker used a valid account to log in.
*   **T1213 — Data from Information Repositories:** The attacker read the draft from the web application.

## What I did NOT do
*   Did not rotate `analyst_yassine`'s password.
*   Did not block the IP.
*   Did not restore `m7-pre-web`.