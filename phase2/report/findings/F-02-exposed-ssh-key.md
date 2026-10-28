# F-02: Exposed SSH Private Key

## Severity
Critical (CVSS 9.8)
CVSS v3.1: AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H

## Affected component
Partner Portal - `/var/www/.ssh/id_ed25519` mounted inside the DVWA container.

## Root Cause
A private SSH key meant for deployment was stored in a web-accessible directory and mounted into a container that was exposed to the DMZ. The key was readable by the `www-data` user.

## Prerequisites for exploitation
A foothold on the Partner Portal (achieved via F-01).

## Evidence
See `phase2/evidence/mission4/key-transfer.txt` and `key-fingerprints.txt`.

## Impact (demonstrated)
Allowed the attacker to pivot from the DMZ into the internal network by authenticating to the Analytics server as the `rivermark` user.

## Remediation
Remove private keys from web servers and containers. Use short-lived credentials, an SSH agent, or a dedicated secrets vault (e.g., HashiCorp Vault).