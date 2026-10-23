# Rivermark Partners: SOC Case Study

## 1. The Firm
Rivermark Partners is a boutique investment research firm. Its crown jewel is not its web server or its SSH ports; it is the **unpublished research drafts**. If these are read by unauthorized parties, the firm loses its competitive edge, client trust, and potentially faces regulatory action.

## 2. The Lab Environment
The environment consists of a Linux portal server (`10.20.30.20`) hosting the web application and a database, accessed by developers and analysts. The server has two network interfaces:
- `rivermark-lan` (10.20.30.0/24): The lab LAN where developers and the analyst connect.
- Management Plane (192.168.56.20): Host-only out-of-band access for the operator (sysadmin).

**Current Hardened State (m10-detection-timer):**
- **Nginx:** Only port 443 (HTTPS) is allowed from the lab LAN. Port 80 redirects to 443.
- **SSH:** Keys only. Passwords are disabled. Connections are restricted to the lab and management subnets via UFW.
- **App:** Runs as `rivermark` user on `127.0.0.1:8080`. Cannot be reached directly from the LAN.
- **File Permissions:** Drafts are `750` (owned by `rivermark`). Even if an attacker gets a shell as a developer, they cannot read the files.

## 3. Normal Activity
Normal activity is a developer (`dev_amine`) logging in via SSH key from `10.20.30.10`, or an analyst (`analyst_yassine`) logging into the web portal from `10.20.30.10`.
*Crucially:* Since both roles use `10.20.30.10`, blocking that IP during an incident would lock out the legitimate staff.

## 4. Incident INC-001: SSH Guessing
An attacker at `10.20.30.10` ran a Hydra brute-force against SSH.
- **What happened:** 20 attempts in 16 seconds. One succeeded (`dev_amine`).
- **Impact:** Low. Because of Unix file permissions, `dev_amine` could not `sudo` and could not read the `/srv/rivermark/drafts/` directory.
- **Evidence:** The server logged 26 lines (PAM checks, failures, successes). The definitive success line was: `Accepted password for dev_amine from 10.20.30.10 port 58990 ssh2`.

## 5. Incident INC-002: Web Brute Force and Data Access
The attacker ran Hydra against the HTTPS portal.
- **What happened:** 4 failed attempts, then a success against `analyst_yassine`. The attacker then used the session cookie to read `draft-001-acme.txt`.
- **Impact:** High. This time, the crown jewel *was* accessed. 
- **Evidence:** Nginx showed `POST /login 200` (failed) and `POST /login 302` (success), but Nginx alone could not identify the *username*. The username was confirmed by joining the Nginx log with the Python app journal, which logged `login success user=analyst_yassine`. 

## 6. Response and Containment
When containing these incidents, we rotated the passwords for `dev_amine` and `analyst_yassine`. We **did not** drop or block `10.20.30.10`, because that address is shared by all legitimate developers and analysts. We chose identity-based containment over network-based containment.

## 7. Hardening's Effect on Detection
We added SSH keys (disabling passwords) and TLS.
- **The SSH Logs:** `Failed password` no longer exists. A password attempt now results in `Connection closed by authenticating user [preauth]`. Our `SSH_GUESS` rule was updated to look for this preauth disconnect, because an attacker trying passwords will generate a burst of these.
- **The Web Logs:** TLS encrypted the traffic, but the HTTP status codes (200 vs 302) remained visible to Nginx. The `WEB_LOGIN_GUESS` rule still works.
- **App Logging:** The Python app was modified to write a `draft_read` line every time a draft is successfully served.

## 8. The Detection Timer
A systemd timer (`rivermark-detect.timer`) runs every 5 minutes. It uses a cursor file (`/var/lib/rivermark-detection/last_run.txt`) so it only reads *new* logs. 
*Bookmark Safety:* If the cursor file is deleted, the script is designed to automatically recreate it using the current time, ensuring old incidents (INC-001 and INC-002) do not trigger a flood of new alerts.
*Limitation:* The detector runs on the same host as the asset. If the server is compromised, the logs can be tampered with.

## 9. Defensible Residuals
- **Stolen Keys:** Since passwords are disabled, an SSH attack now requires a stolen private key.
- **Online Web Guessing:** TLS hides the password in transit, but does not prevent an attacker from guessing the password online.
- **No MFA:** We refused to implement MFA to avoid locking out the `sysadmin` without a safe console-recovery plan.
- **contractor_old:** Kept locked and present. It is a detection tripwire.

---

## Task C: Spoken Answers

**1. Why did a successful SSH as dev_amine not mean the research was stolen?**
While the attacker gained a valid shell on the host (which is an impact in itself), they did not steal the research. The file mode (750) blocked `dev_amine` from reading `/srv/rivermark/drafts/`. I proved this by actively trying to `cat` the file as `dev_amine` and receiving a Permission denied error. Furthermore, the nginx access log showed zero draft requests during the SSH window, confirming the shell was never used to access the web portal. 

**2. Why did the web threshold move from 5 to 3, and what false positive did you accept?**
The threshold was lowered because Hydra only generated 4 failed logins before finding the correct password. A threshold of 5 would have missed the breach. By lowering it to 3, we accept the false positive that an analyst who mistypes their password three times in one minute will trigger an alert. However, this is an overfit lab rule: because we only had one typo in the baseline and 4 failures in the attack, 3 was the only number that worked. I would not ship this threshold to a production firm without a full week of real analyst behavior to establish a true baseline.

**3. Why is "Connection closed by authenticating user [preauth]" not the same incident as "Failed publickey"?**
`Failed publickey` means a key was offered and rejected by the server. `Connection closed... [preauth]` means the connection died before authentication even completed. Hydra trying passwords on a key-only server generates the preauth close. Treating a rejected key as a password spray means you are alerting on the wrong attack vector.