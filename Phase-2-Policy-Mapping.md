# Phase 2 Policy Mapping

- **Time estimate:** 15–20 minutes
- **Goal:** Identify which policy statements require technical proof and map each one to the event data that proves it.
- No tools required this is a reading and mapping exercise.

← [Back to Index](README.md)

---

## Why This Phase Exists

The most common audit weakness is a gap between what a policy *says* and what the system *does*. This phase forces you to read the actual policy language before generating any data. If you generate logs before understanding the controls, you end up with data that doesn't map cleanly to a specific policy commitment.

Every query you write in Phase 4 directly implements one row of the mapping table at the bottom of this document.

---

## Source Policy

**Policy document:** `[FILL: your-policy-document-name]` (e.g. your Logging and Monitoring Policy from a prior ISMS lab)

Open that document. The relevant sections should cover:
- Required log sources
- Log integrity requirements
- Monitoring and alerting thresholds

> **Note:** If you don't have a prior policy document, create a simple one-page policy that defines at minimum: what events you will collect, a failure threshold that triggers an alert (e.g. 10 failed logins/24h), and a commitment to logging admin account changes.

---

## Control 1: AC-7 Login Attempt Limits

**NIST SP 800-53 Control:** AC-7 requires that the system enforce a limit on consecutive invalid login attempts and lock or delay access when that limit is exceeded.

**CSF 2.0 Function:** DE.AE-02 Potentially adverse events are analyzed to better characterize them.

> **Policy Statement ([FILL: your-policy-document], Section [FILL: section]):**
> "[FILL: paste the relevant policy language here — e.g. 'Failed login attempts from a single IP: 10 failures in 24 hours triggers an automated alert']"

**What this threshold means technically:**

The policy commits to detecting brute force by counting Event ID **4625** (failed login) per source IP. The threshold you define is the trigger for a dashboard alert. The Splunk dashboard proves this detection is operational, not theoretical.

**Event Data Required:**

| Windows Event ID | Meaning | Standard Field Names |
|---|---|---|
| 4625 | Failed logon | `EventCode`, `IpAddress`, `SubjectUserName`, `_time` |

**What the SPL Query Must Prove:**

Count of 4625 events grouped by `IpAddress` over your defined time window. A source IP at or above your threshold satisfies the detection requirement.

> **Audit statement you can use:**
> "AC-7 is operationalized: failed logins are collected via HEC, counted in Splunk, and any IP exceeding [FILL: your threshold] failures in [FILL: your time window] is surfaced in the dashboard. This directly implements [FILL: policy section]."

---

## Control 2: AU-9 Protection of Audit Information

**NIST SP 800-53 Control:** AU-9 requires that the system protect audit information and tools from unauthorized access, modification, and deletion.

**CSF 2.0 Function:** DE.CM-03 Personnel activity and technology usage are monitored to find potentially adverse events.

> **Policy Statement ([FILL: your-policy-document], Section [FILL: section]):**
> "[FILL: paste the relevant policy language here — e.g. 'New admin account created outside change window: Any occurrence triggers immediate investigation']"

**What this means technically:**

The policy commits to logging all admin account creation and modification events. Event IDs **4720** (account created) and **4722** (account enabled) are the triggers. Centralizing these in Splunk and displaying them in a dashboard panel proves the SIEM is tracking admin account lifecycle changes in real time which is the audit trail AU-9 requires.

**Event Data Required:**

| Windows Event ID | Meaning | Standard Field Names |
|---|---|---|
| 4720 | User account created | `EventCode`, `SubjectUserName`, `TargetUserName`, `_time` |
| 4722 | User account enabled | `EventCode`, `SubjectUserName`, `TargetUserName`, `_time` |

**What the SPL Query Must Prove:**

All 4720 and 4722 events in the last 7 days, displayed in a table with timestamp, who performed the action (`SubjectUserName`), and which account was affected (`TargetUserName`).

> **Audit statement you can use:**
> "AU-9 is operationalized: all admin account creation and modification events are captured via HEC and displayed in the Admin Activity Audit panel. Any account creation is immediately visible to the SOC, implementing the real-time detection requirement in [FILL: policy section]."

---

## Control 3: DE.CM-01 Continuous Authentication Monitoring

**NIST SP 800-53 Controls:** SI-4 (System Monitoring), AU-2 (Event Logging)

**CSF 2.0 Function:** DE.CM-01 Networks and services are monitored to find potentially adverse events.

> **Policy Statement ([FILL: your-policy-document], Section [FILL: section]):**
> "[FILL: paste the relevant policy language here e.g. 'SIEM ingests logs from all in-scope sources in real time and correlates events to identify attack patterns']"

**What this means technically:**

This is the broadest of the three controls. It requires that both successful (4624) and failed (4625) logins are collected and visible over time. A trend chart showing success vs. failure volume over 7 days proves that continuous monitoring is active, not just snapshot-based.

**Event Data Required:**

| Windows Event ID | Meaning | Standard Field Names |
|---|---|---|
| 4624 | Successful logon | `EventCode`, `SubjectUserName`, `IpAddress`, `_time` |
| 4625 | Failed logon | `EventCode`, `SubjectUserName`, `IpAddress`, `_time` |

**What the SPL Query Must Prove:**

Counts of 4624 and 4625 events grouped by day over 7 days, displayed as a line chart. A baseline of successful authentications with a visible spike in failures when brute force events are present proves the monitoring captures both normal and anomalous activity.

> **Audit statement you can use:**
> "DE.CM-01 is operationalized: both successful and failed authentication events are ingested in real time and visualized as a 7-day trend. Any anomalous spike in failed logins is immediately visible against the baseline of successful authentications."

---

## Controls Deferred From This Lab

Document any policy commitments you are intentionally not proving in this lab, and why. This shows an auditor that the scope boundary is deliberate, not a gap.

| Control | Policy Statement | Reason Deferred | 
|---|---|---|
| AC-2 (After-Hours Privileged Access) | Section 3.6: Privileged access outside business hours requires alerting and approval | Requires timezone-aware SPL queries and admin role classification. Needs live directory integration | 
| AU-5 (Log Forwarding Validation) | Section 3.5: All security logs must forward to SIEM with heartbeat verification | Cannot simulate live forwarder without additional infrastructure. Requires actual Windows Event Forwarder or syslog agent. |

> These are not gaps in the policy. They are commitments that require a more complex technical environment than a single-user local SIEM lab provides.

---

## Complete Control Mapping Table

This is the full traceability from policy to technical proof. Every row maps directly to a dashboard panel in Phase 5.

| SP 800-53 Control | CSF 2.0 Function | Event ID | Standard Fields | Detection Logic | Dashboard Panel |
|---|---|---|---|---|---|
| AC-7 | DE.AE-02 | 4625 | `IpAddress`, `SubjectUserName` | Count failures/IP, last 24h alert if ≥ [FILL: threshold] | Panel 1: Single value |
| AC-7 | DE.AE-02 | 4625 | `IpAddress`, `SubjectUserName` | Top IPs by failure count, last 24h | Panel 2: Brute Force table |
| AU-9 | DE.CM-03 | 4720, 4722 | `SubjectUserName`, `TargetUserName` | All account creation/enable events, last 7d | Panel 3: Admin Activity table |
| DE.CM-01 | DE.CM-01 | 4624, 4625 | `IpAddress`, `SubjectUserName` | Success vs. failure trend by day, last 7d | Panel 4: Line chart |

---

## Required Fields for Phase 3

Before generating data, confirm your Python script will produce these fields so the SPL queries in Phase 4 work correctly.

**Required fields in every event (Windows standard naming):**

| Field | Windows Standard Name | Purpose | Example Value |
|---|---|---|---|
| Event type | `EventCode` | Drives all four queries | `4625` |
| Source IP | `IpAddress` | AC-7 grouping | `[FILL: test IP, e.g. 203.0.113.45]` |
| Actor (who did it) | `SubjectUserName` | AU-9 actor, auth trends | `[FILL: e.g. svc-admin]` |
| Target (who was affected) | `TargetUserName` | AU-9 target account | `[FILL: e.g. newuser01]` |
| Timestamp | `_time` | All time-based queries | Unix timestamp |
| Description | `Message` | Human-readable context | `[FILL: e.g. "Account 'X' was created by 'Y'"]` |

**Why standard Windows field names?**
Splunk extracts these automatically when `sourcetype` is `_json`. Using standard names means your SPL queries transfer directly to a real enterprise Windows environment — no remapping required.

---

## Phase 2 Checkpoint

- [ ] You can cite which policy section commits to your failure threshold
- [ ] You know which Event IDs map to each control (4625 → AC-7, 4720/4722 → AU-9, 4624/4625 → DE.CM-01)
- [ ] You understand why the trend chart (Panel 4) proves DE.CM-01 rather than AC-7
- [ ] You know what standard Windows field names your script must include
- [ ] You have documented any deferred controls with a clear reason

---

## Next Phase

[→ Phase 3: Data Generation](Phase-3-Data-Generation.md)
