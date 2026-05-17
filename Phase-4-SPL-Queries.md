---
title: Phase 4 - SPL Queries
lab_id: "[FILL: Your Lab ID]"
phase: 4
type: lab-phase
status: complete
created: "[FILL: YYYY-MM-DD]"
tags: [splunk, spl, lab]
---

**Time estimate:** 30–45 minutes
**Goal:** Build and test all four SPL queries in Splunk Search. Each query must return the expected output before you move to Phase 5.

← [Back to Lab Index](README.md)

---

## How to Use This Phase

For each query:
1. Open Search & Reporting in the Splunk UI (`http://localhost:8000`)
2. Paste the query into the search bar
3. Set the time range as specified
4. Run it and compare your output to the expected result
5. Check the box in the checkpoint at the end when it matches

Do not move to Phase 5 until all four queries return expected results. The dashboard in Phase 5 is built directly from these queries — if they are broken, the panels will be empty.

---

## Query 1: Failed Logins Count (AC-7)

**NIST Control:** AC-7 — Login Attempt Limits
**Policy Reference:** [FILL: your policy section] — e.g. "10 failed logins from a single IP in 24 hours triggers an alert"
**Dashboard Panel:** Panel 1 — Failed Logins Single Value

**Time range:** Last 24 hours

```spl
index=[FILL: your-index-name] EventCode=4625
| stats count AS failed_logins
```

**Expected output:**

A single number representing total failed login events in the last 24 hours. This number should exceed your policy threshold (e.g. 10) to demonstrate the detection is active.

[INSERT SCREENSHOT: Splunk single value query result]

> **Why this matters:**
> This is the number displayed on the single-value panel. When it exceeds your threshold, the panel turns red — proving the threshold defined in your policy is being actively monitored.

---

## Query 2: Brute Force Source Table (AC-7)

**NIST Control:** AC-7 — Login Attempt Limits
**Policy Reference:** [FILL: your policy section] — detect brute force by source IP
**Dashboard Panel:** Panel 2 — Brute Force Sources Table

**Time range:** Last 7 days

```spl
index=[FILL: your-index-name] EventCode=4625
| stats count AS failures BY IpAddress
| sort - failures
| where failures >= [FILL: your threshold, e.g. 10]
| rename IpAddress AS "Source IP", failures AS "Failed Attempts"
```

**Expected output:**

| Source IP | Failed Attempts |
|---|---|
| [FILL: your-test-ip] | ~[FILL: expected count] |

Only IPs at or above your threshold should appear. All others are filtered by the `where` clause.

[INSERT SCREENSHOT: Brute force source table]

**If no results appear:**
- Remove the `where` clause temporarily to see all IPs and their counts
- Confirm your test IP is in the data: `index=[FILL: your-index-name] EventCode=4625 IpAddress="[FILL: your-test-ip]"`
- If it is missing, re-run `generate_logs.py` (Phase 3)

> **The `rename` commands** make column headers human-readable for the dashboard. "Source IP" reads better than `IpAddress` in an audit presentation.

---

## Query 3: Admin Activity Audit (AU-9)

**NIST Control:** AU-9 — Protection of Audit Information
**Policy Reference:** [FILL: your policy section] — e.g. "New admin account created outside change window triggers immediate investigation"
**Dashboard Panel:** Panel 3 — Admin Activity Audit Table

**Time range:** Last 7 days

```spl
index=[FILL: your-index-name] (EventCode=4720 OR EventCode=4722)
| eval event_type=case(EventCode="4720", "Account Created", EventCode="4722", "Account Enabled", true(), "Unknown")
| table _time, event_type, SubjectUserName, TargetUserName, Message
| sort - _time
| rename _time AS "Timestamp", event_type AS "Event Type", SubjectUserName AS "Performed By", TargetUserName AS "Target Account", Message AS "Details"
```

**Expected output ([FILL: expected row count] rows):**

| Timestamp | Event Type | Performed By | Target Account | Details |
|---|---|---|---|---|
| [FILL: date] | Account Created | [FILL: actor] | [FILL: target] | [FILL: message] |
| [FILL: date] | Account Enabled | [FILL: actor] | [FILL: target] | [FILL: message] |

**If fewer rows appear than expected:**
- Check the time range — expand to "All time" to confirm events exist
- Run `index=[FILL: your-index-name] (EventCode=4720 OR EventCode=4722)` with "All time" to verify

> **`eval` + `case` pattern:**
> `eval event_type=case(EventCode="4720", ...)` translates numeric Event IDs into readable labels. In a real SOC dashboard, "Account Created" is more useful to an analyst than "4720." This is a standard SPL pattern worth remembering.

---

## Query 4: Authentication Trends (DE.CM-01)

**NIST Control:** DE.CM-01 — Continuous Monitoring, DE.AE-02
**Policy Reference:** [FILL: your policy section] — e.g. "SIEM ingests logs in real time and correlates events across sources"
**Dashboard Panel:** Panel 4 — Authentication Trends Line Chart

**Time range:** Last 7 days

```spl
index=[FILL: your-index-name] (EventCode=4624 OR EventCode=4625)
| eval auth_result=case(EventCode="4624", "Success", EventCode="4625", "Failure", true(), "Unknown")
| timechart span=1d count BY auth_result
```

**Expected output:**

One row per day with two columns — `Success` and `Failure`. The day you ran the bulk data generation should show a spike in failures. Earlier days should show normal ratios.

| _time | Failure | Success |
|---|---|---|
| [FILL: date -6d] | [FILL: count] | [FILL: count] |
| [FILL: date -5d] | [FILL: count] | [FILL: count] |
| ... | ... | ... |
| [FILL: today] | [FILL: high count] | [FILL: count] |

[INSERT SCREENSHOT: Authentication trends timechart output]

**If the chart shows only one day:**
- Check historical events actually landed: `index=[FILL: your-index-name] earliest=-7d@d | stats count by date_mday`
- If you see only one date, re-run `generate_logs.py` — the timestamp spread may not have worked correctly

> **`timechart` vs `stats`:**
> `timechart` is Splunk's built-in time-bucketing command. `span=1d` groups events by day and automatically creates one column per value of the `BY` field. Use `timechart` for any trend visualization.

---

## Phase 4 Checkpoint

- [ ] Query 1 returns a count above your alert threshold for the last 24 hours
- [ ] Query 2 returns your designated brute-force IP with a high failure count
- [ ] Query 3 returns the expected number of account creation and account enable events
- [ ] Query 4 returns a 7-day table with a visible spike on the data generation day

**All four passing?** Move to Phase 5.

---

## SPL Quick Reference

Patterns from this lab worth retaining:

| Pattern | What It Does |
|---|---|
| `stats count BY field` | Count events grouped by a field value |
| `sort - fieldname` | Sort descending by a field |
| `where fieldname >= N` | Filter rows after aggregation (equivalent to SQL HAVING) |
| `eval field=case(condition, value, ...)` | Conditional field assignment |
| `timechart span=1d count BY field` | Time-bucketed count by field value |
| `rename old AS "New Label"` | Rename a field for display |
| `table field1, field2, ...` | Select and order output columns |

---

## Next Phase

[→ Phase 5: Dashboard](Phase-5-Dashboard.md)
