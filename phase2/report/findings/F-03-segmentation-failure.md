# F-03: Network Segmentation Failure

## Severity
High (CVSS 8.8)
CVSS v3.1: AV:A/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H

## Affected component
Partner Portal VM - dual-homed interface configuration.

## Root Cause
The Partner Portal VM has interfaces on both `10.20.40.0/24` and `10.20.30.0/24` and forwards traffic between them without firewall restrictions (CWE-923: Improper Restriction of Communication Channel). This is a network architecture design decision, not a bug.

## Prerequisites
A shell on the Portal (achieved via F-01).

## Evidence
See `phase2/report/evidence-redacted/kali-nmap-from-tunnel.txt`.

## Impact (demonstrated)
The pivot from DMZ to internal was possible because the Portal forwarded traffic. The attack chain would have terminated at the Portal if this had not been the case.

## Remediation
- **Symptom Fix:** Add a host firewall rule on the Portal that denies all forwarded traffic to `10.20.30.0/24`.
- **Root Cause Fix:** Remove the second NIC from the Portal unless a documented business need exists; isolate it into a dedicated jump host with strict ACLs.

## Retest procedure
From a DMZ position, attempt `nc -vz 10.20.30.40 22`. Connection should time out.