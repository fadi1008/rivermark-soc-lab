# Scope and Rules of Engagement

## In-Scope Targets
- **Partner Portal (DMZ):** `10.20.40.20`
- **Analytics Server (Internal):** `10.20.30.40`
- **Attack Platform:** `10.20.40.10` (Kali Linux)

## Out-of-Scope
- The Phase 1 `rivermark-portal` (powered off).
- Denial of Service (DoS) attacks.
- Physical security testing.

## Rules of Engagement
- Testing was performed within the isolated lab environment.
- No data was exfiltrated outside the lab network.
- Root access was obtained and confirmed, but no persistence mechanisms (cron, user creation) were left on the target.