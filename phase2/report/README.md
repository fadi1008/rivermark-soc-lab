# Rivermark Partners — Phase 2 Penetration Test Report

**Engagement Type:** External to Internal Network Penetration Test
**Date of Assessment:** October 2026
**Classification:** INTERNAL / PORTFOLIO (Sanitized)
**Author:** Security Analyst

---

## ⚠️ Disclaimer & Ethical Statement
This report documents a simulated penetration test conducted within a controlled, isolated virtual lab environment. "Rivermark Partners" is a fictional company. All IP addresses belong to RFC 1918 private ranges. All data (including the "confidential research drafts") is synthetic. No real-world systems, networks, or individuals were harmed or accessed during this engagement. 

This document has been sanitized for portfolio purposes. Sensitive material, including raw private keys and active session tokens, have been redacted or omitted. 

---

## 📖 Report Navigation

This report is structured to be read by both technical and non-technical stakeholders. If you are short on time, start with the **Executive Summary**.

### 1. Executive Overview
- [Executive Summary](./00-executive-summary.md) — *Business impact and prioritized recommendations.*
- [Scope and Rules of Engagement](./01-scope-roe.md) — *What was tested and the rules of the engagement.*

### 2. Technical Analysis
- [Methodology and Limitations](./02-methodology.md) — *Tools used, testing approach, and unexplored hypotheses.*
- [Attack Narrative](./03-attack-narrative.md) — *The chronological story of the breach, from recon to root.*
- [Attack Chain Diagram](./04-attack-chain-diagram.md) — *Visual representation of the DMZ-to-Internal pivot.*
- [Detection Gap Analysis](./06-detection-gaps.md) — *How the attack appeared (or didn't appear) to the Phase 1 SOC.*

### 3. Findings
- [**F-01: Command Injection in DVWA**](./findings/F-01-command-injection.md) — *(Critical)*
- [**F-02: Exposed SSH Private Key**](./findings/F-02-exposed-ssh-key.md) — *(Critical)*
- [**F-03: Network Segmentation Failure**](./findings/F-03-segmentation-failure.md) — *(High)*
- [**F-04: Privilege Escalation via Sudo Vim**](./findings/F-04-sudo-vim.md) — *(High)*

### 4. Appendices
- [Attack Timeline](./07-timeline.md) — *Verified timestamps of the attack chain.*
- [MITRE ATT&CK Mapping](./08-attack-mapping.md) — *Tactics and techniques observed.*
- [Remediation Plan](./09-remediation.md) — *Prioritized fix list for the client.*
- [Retest Plan](./10-retest-plan.md) — *Steps to verify fixes (includes verified retest for F-04).*

---

## 📁 Evidence
All claims made in this report are backed by sanitized evidence files located in the `evidence-redacted/` directory. The original raw evidence remains in the secure internal lab environment.

## 🏆 Quality Gate Status
- [x] Level 1: Basic Walkthrough
- [x] Level 2: Professional Report (12/12 criteria met)
- [x] Level 3: Expert Analysis (17/17 criteria met, including verified remediation retest)