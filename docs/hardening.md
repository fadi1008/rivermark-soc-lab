## Automated Detection (Mission 10)

- **Mechanism:** A systemd timer (`rivermark-detect.timer`) runs `auto_detect.py` every 5 minutes.
- **Cursor:** It uses `/var/lib/rivermark-detection/last_run.txt` to only query new logs. Old incidents will not re-alert. 
- **Output:** Alerts are appended to `/var/log/rivermark-alerts.log`.
- **Exit Status:** The service exits `1` when an alert is found (triggering a systemd warning), and `0` when quiet. `SuccessExitStatus=1` is set in the unit so systemd doesn't treat a successful detection as a failure.
- **SSH Rule Update:** Since passwords are disabled, `SSH_GUESS` now looks for `Connection closed by authenticating user [preauth]` instead of `Failed password`.
- **Limitations:** This runs on the same server that hosts the app. If the server is compromised, the attacker can edit the logs and the alert file. We will address this with a SIEM/centralized logging in a later phase.
- **Residuals:** No Fail2ban, no IP bans. TLS does not stop online password guessing against `WEB_LOGIN_GUESS`.