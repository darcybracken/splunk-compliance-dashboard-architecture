# NIST Control Validation with Splunk

A hands-on lab demonstrating how to operationalize NIST SP 800-53 and CSF 2.0 security controls using a locally hosted Splunk SIEM. I built a Python-based log generator, ingested 500 synthetic Windows Event Log events via HEC, wrote SPL queries mapped to specific policy thresholds, and shipped a four-panel compliance dashboard that gives an auditor live proof that controls AC-7, AU-9, and DE.CM-01 are active.

> "The difference between saying a control exists and proving it."

---

## What This Lab Proves

| Control | Framework | What It Detects | Dashboard Panel |
|---|---|---|---|
| AC-7 | NIST SP 800-53 | Brute force IPs with 10+ failed logins/24h | Panel 1 (single value), Panel 2 (table) |
| AU-9 | NIST SP 800-53 | Admin account creation and enable events | Panel 3 (table) |
| DE.CM-01 | NIST CSF 2.0 | 7-day authentication success vs. failure trend | Panel 4 (line chart) |

---

## Dashboard Screenshot

screenshots/nist-compliance-dashboard.png <img width="1066" height="742" alt="nist-compliance-dashboard" src="https://github.com/user-attachments/assets/ac234b66-58d2-4479-90c9-57aa0bb6f69e" />


---

## Repository Structure

```
Splunk-NIST-Validation/
├── FINDINGS.ED
├── LESSONS-LEARNED.md                ← Every failure, why it happened, how I fixed it
├── Phase-1-Environment-Setup.md      ← Splunk install, HEC config, dead ends
├── Phase-2-Policy-Mapping.md         ← Control → Event ID → SPL traceability
├── Phase-3-Data-Generation.md        ← generate_logs.py walkthrough and verification
├── Phase-4-SPL-Queries.md            ← All four queries with expected output
├── Phase-5-Dashboard.md              ← XML import, troubleshooting, XML breakdown
├── Phase-6-Documentation-Closure.md  ← Traceability matrix, audit chain walkthrough
└── README.md                         ← You are here

```

---

## Traceability Matrix

| Control ID | Framework | Event ID | Meaning | SPL Logic | Panel | 
|---|---|---|---|---|---|
| AC-7 | NIST SP 800-53 | 4625 | Failed Logon | `stats count AS failed_logins` | Panel 1 |
| AC-7 | NIST SP 800-53 | 4625 | Failed Logon by Source | `stats count BY IpAddress \| where >= 10` | Panel 2 | 
| AU-9 | NIST SP 800-53 | 4720 | Account Created | `table Timestamp, SubjectUserName, TargetUserName` | Panel 3 | 
| AU-9 | NIST SP 800-53 | 4722 | Account Enabled | `table Timestamp, SubjectUserName, TargetUserName` | Panel 3 | 
| DE.CM-01 | NIST CSF 2.0 | 4624, 4625 | Auth Trend | `timechart span=1d count BY auth_result` | Panel 4 |
| DE.AE-02 | NIST CSF 2.0 | 4624, 4625 | Success/Failure Delta | `timechart span=1d count BY auth_result` | Panel 4 | 

---

## Tech Stack

| Tool | Purpose |
|---|---|
| Splunk Enterprise | SIEM event ingestion, search, dashboard |
| Python 3 | Log generator (`generate_logs.py`) |
| SPL (Splunk Processing Language) | Query and dashboard logic |
| HEC (HTTP Event Collector) | REST-based event ingestion |
| NIST SP 800-53 / CSF 2.0 | Control framework |

---

## SPL Quick Reference

| Pattern | What It Does |
|---|---|
| `stats count BY field` | Count events grouped by a field value |
| `sort - fieldname` | Sort descending |
| `where fieldname >= N` | Filter after aggregation (SQL HAVING equivalent) |
| `eval field=case(condition, value, ...)` | Conditional field assignment |
| `timechart span=1d count BY field` | Time-bucketed count by field value |
| `strftime(_time, "%Y-%m-%d %H:%M:%S")` | Convert Unix timestamp to readable date |

---

## Key Lessons Learned

Eight significant failures hit during this lab every one of them documented with root cause and fix. The biggest:

- Splunk's HEC setup form pre-fills the wrong port (`8000` instead of `8088`) and saves without error the misconfiguration is completely silent until you test it
- A successful HTTP 200 from HEC does not mean fields were extracted verify with `stats count by EventCode` in Verbose Mode, not Smart Mode
- Stale events from failed runs accumulate in the index and will corrupt test results — always stop Splunk, clean, and restart before a fresh run

Full details in [LESSONS-LEARNED.md](LESSONS-LEARNED.md).

---

## Platform

Built on macOS (Apple Silicon M4). Splunk installed natively via DMG Docker does not support ARM64 Splunk images. The HEC configuration and SPL queries are platform-agnostic.

---

## Related Work

- [FILL: link to your ISMS lab or policy document if public]
- [My Linkedin](www.linkedin.com/in/darcyvbracken)
