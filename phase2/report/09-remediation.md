# Remediation Plan

## Priority 1 — Immediate (within 7 days)

| Finding | Remediation | Estimated Effort | Owner |
|---|---|---|---|
| F-02 | Rotate the exposed key; move it out of `/var/www`; add CI check | 1 day | AppSec + Platform |
| F-04 | Remove NOPASSWD rule; replace with wrapper; enforce sudoers review | 2 days | Platform + Security |
| F-03 | Remove second NIC from Portal, or add explicit deny on forwarded traffic | 1 day | Network |

## Priority 2 — Short term (within 30 days)
| Finding | Remediation | Estimated Effort | Owner |
|---|---|---|---|
| F-01 | Design review, input validation, escapeshellarg() | 3 days | AppSec |

## Priority 3 — Structural (within 90 days)
- Implement a sudoers CI/CD lint rule.
- Implement secret-scanning in CI.
- Review network segmentation as part of change control.