# F-02: Exposed SSH Private Key

## Severity
Critical (CVSS 9.8)
*Note: The chain from F-01 to F-04 is Critical because each finding is a precondition of the next, terminating in root on a data-bearing host.*
CVSS v3.1: AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H

## Affected component
Partner Portal - `/var/www/.ssh/id_ed25519` mounted inside the DVWA container.

## Root Cause
A private SSH key was stored in a web-accessible directory and mounted into a container exposed to the DMZ (CWE-522: Insufficiently Protected Credentials). The key was readable by the `www-data` user.

## Prerequisites
A foothold on the Partner Portal (achieved via F-01).

## Evidence
See `phase2/report/evidence-redacted/key-transfer-redacted.txt`.

## Impact (demonstrated)
Allowed the attacker to pivot from the DMZ into the internal network, authenticating to the Analytics server as the `rivermark` user.

## Remediation
- **Symptom Fix:** Rotate the key immediately.
- **Root Cause Fix:** Move key storage to a proper secret store (e.g., HashiCorp Vault). Add CI check that fails the build if any file under `/var/www` matches `id_*`.

## Retest procedure
1. `sudo -u www-data find /var/www -name 'id_*'` returns nothing.
2. The old key is refused by the target.