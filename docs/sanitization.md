# Repository Sanitization

Before finalizing this project, a search was conducted across the Git history to ensure no live secrets were committed.

**What was searched:**
- Old and new passwords (`369258147`, `NewDevPass2026!`, `NewWebPass2026!`)
- Private keys (`BEGIN OPENSSH PRIVATE KEY`)
- Active session cookies (`session=`)

**Result:**
The repository contains no live secrets, private keys, or active session cookies. All fixtures in the `evidence/` folder contain raw log lines and HTTP requests with placeholder passwords (e.g., `password=wrong`). No actual credentials were committed. The `sysadmin`, `dev_amine`, and `analyst_yassine` passwords remain only in local, out-of-band credential managers.