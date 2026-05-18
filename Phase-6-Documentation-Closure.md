
# Phase 6 Documentation Closure

- **Goal:** Finalize evidence, verify end-to-end traceability.

← [Back to Index](README.md)


## What This Phase Does

This phase closes the loop between your policy documentation and the technical implementation.


## Step 1 Update Your Policy Document

If you have a prior policy document (e.g. from an ISMS Doc), open it and update the technical validation section.

Add the following to your policy's evidence section:

## Technical Validation

**Screenshot Reference:**

! NIST Compliance Dashboard
<img width="1080" height="1033" alt="image" src="https://github.com/user-attachments/assets/c8b111a2-776c-4e02-a21f-793571a2f31d" />


* Dashboard shows all four panels:
Failed Logins (AC-7), Brute Force Source (AC-7), Admin Activity Audit (AU-9),
and Authentication Trends (DE.CM-01). Single value panel shows [FILL: COUNT]
failures in the last 24 hours — above the [FILL: THRESHOLD]-failure alert
threshold defined in [FILL: policy section].*

---

## Step 2 Walkable Audit Chain

**For AC-7 (Brute Force Detection):**

```
Policy Mandate          Framework Mapping       Ingestion Contract
policy section          NIST SP 800-53 AC-7     Windows Event ID 4625
threshold/24h           CSF 2.0 DE.AE-02        Structured JSON via HEC

         |
         v

Presentation Layer      Analytics Pipeline      Parsing Engine
Dashboard Panel 2       SPL Filter Constraint   Automatic Key Extraction
"Brute Force Sources"   where failures >= N     via _json Sourcetype
```

**For AU-9 (Admin Account Monitoring):**

```
Policy Mandate          Framework Mapping       Ingestion Contract
policy section          NIST SP 800-53 AU-9     Windows Event IDs 4720, 4722
Any account creation    CSF 2.0 DE.CM-03        Structured JSON via HEC
triggers investigation

         |
         v

Presentation Layer      Analytics Pipeline      Parsing Engine
Dashboard Panel 3       eval + table + sort     Automatic Key Extraction
"Admin Activity Audit"  SubjectUserName,        via _json Sourcetype
                        TargetUserName
```

---

## Traceability Matrix

This is the complete audit-ready mapping from policy to technical proof.

| **Control ID** | **Regulatory Framework** | **Target Event ID** | **System Meaning**      | **Underlyling SPL Search Logic**                           | **Active Dashboard Target**      | **Policy Reference** |
| -------------- | ------------------------ | ------------------- | ----------------------- | ---------------------------------------------------------- | -------------------------------- | -------------------- |
| **AC-7**       | NIST SP 800-53           | 4625                | Failed Logon            | `stats count AS failed_logins`                             | Panel 1: Single Value Tracker    | FC-POL-002 §3.4      |
| **AC-7**       | NIST SP 800-53           | 4625                | Failed Logon by Source  | `stats count BY IpAddress \| where failures >= 10`         | Panel 2: Brute Force Audit Table | FC-POL-002 §3.4      |
| **AU-9**       | NIST SP 800-53           | 4720                | Account Object Creation | `table _time, event_type, SubjectUserName, TargetUserName` | Panel 3: Privilege Action Audit  | FC-POL-002 §3.4      |
| **AU-9**       | NIST SP 800-53           | 4722                | Account Control Enabled | `table _time, event_type, SubjectUserName, TargetUserName` | Panel 3: Privilege Action Audit  | FC-POL-002 §3.4      |
| **DE.CM-01**   | NIST CSF 2.0             | 4624, 4625          | Authentication Trend    | `timechart span=1d count BY auth_result`                   | Panel 4: 7-Day Line Chart        | FC-POL-002 §3.5      |
| **DE.AE-02**   | NIST CSF 2.0             | 4624, 4625          | Success/Failure Delta   | `timechart span=1d count BY auth_result`                   | Panel 4: 7-Day Line Chart        | FC-POL-002 §3.4      |

---

## Phase 6 Checkpoint

- [ ] Dashboard screenshot saved
- [ ] Policy document update (if applicable)
- [ ] Traceability matrix verified all rows accurate
- [ ] Audit chain walkable from memory for AC-7
- [ ] Audit chain walkable from memory for AU-9

