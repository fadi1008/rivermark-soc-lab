# Web Detection Rules

## Rule 1 — WEB_LOGIN_GUESS
**Intent:** Detect brute force against the portal login form.
**Threshold:** 3 failed POST /login (HTTP 200) from one IP within 60 seconds. One alert per source IP per window.
**Why 3:** 
- Baseline: A normal user typing a password fails 1 time (a typo).
- INC-002: Hydra fired 5 passwords. 4 failed, 1 succeeded. (The initial threshold of 5 missed the attack because Hydra found the password on the 5th try).
- 3 is above the baseline, but low enough to catch a short brute-force burst before the successful guess.

## Rule 2 — WEB_DRAFT_AFTER_GUESS
**Intent:** Detect successful exploitation of a guessed credential.
**Threshold:** A GET to `/drafts/*` returning HTTP 200 from an IP that triggered WEB_LOGIN_GUESS within the last 60 seconds.
**Critical Detail:** We join nginx logs (to see the draft read) with the app journal (to get the username `analyst_yassine`), because nginx does not log POST bodies.