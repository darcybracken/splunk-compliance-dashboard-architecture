---
title: Phase 3 - Data Generation
lab_id: "[FILL: Your Lab ID]"
phase: 3
type: lab-phase
status: complete
created: "[FILL: YYYY-MM-DD]"
tags: [splunk, python, lab]
---

**Time estimate:** 20–30 minutes
**Goal:** Run `generate_logs.py` and confirm all events appear in your Splunk index.

← [Back to Lab Index](README.md)

---

## What the Script Does

`generate_logs.py` generates synthetic Windows Event Log events and sends them directly to Splunk via HEC. There are no intermediate log files — the script POSTs JSON directly to `http://localhost:8088`.

**Event breakdown:**

| Event Type | Event ID | Count | Purpose |
|---|---|---|---|
| Brute force failures (single IP) | 4625 | [FILL: e.g. 50] | Triggers AC-7 alert — one IP exceeds threshold |
| Normal failed logins (multiple IPs) | 4625 | [FILL: e.g. 100] | Background noise — spreads failures across many IPs |
| Admin account creations | 4720 | [FILL: e.g. 5] | AU-9 — account creation audit trail |
| Admin account enables | 4722 | [FILL: e.g. 5] | AU-9 — account enable audit trail |
| Privileged logins | 4624 | [FILL: e.g. 10] | Admin logins mixed into success data |
| Normal successful logins | 4624 | [FILL: e.g. 230] | DE.CM-01 baseline data |
| Historical events (7-day spread) | 4624, 4625 | [FILL: e.g. 100] | Fills the trend chart with data across 7 days |
| **Total** | | **[FILL: total]** | |

---

## Step 1: Get the Script

`generate_logs.py` is included in this repository. Copy it to your working directory:

```bash
cp /path/to/repo/generate_logs.py ~/[FILL: your-working-directory]/
```

Or confirm it exists:

```bash
ls -lh ~/[FILL: your-working-directory]/generate_logs.py
```

---

## Step 2: Verify Your Configuration

Open `generate_logs.py` and confirm these values match your Phase 1 setup:

```python
SPLUNK_HOST = "http://localhost:8088"
HEC_TOKEN   = "[YOUR-TOKEN-UUID]"
INDEX       = "[FILL: your-index-name]"
SOURCETYPE  = "_json"
```

To find your token UUID in Splunk:

```
Settings → Data Inputs → HTTP Event Collector → click the token name → copy the Token Value field
```

> **Four things must be correct before running:**
> - `SPLUNK_HOST` must use port `8088` — not `8089` (management API) or `8000` (web UI)
> - `HEC_TOKEN` must be the UUID token value, not the token name
> - `INDEX` must match exactly — it is case-sensitive
> - `SOURCETYPE` must be `_json` — without this, Splunk accepts events but does not extract fields

---

## Step 3: Run the Script

```bash
cd ~/[FILL: your-working-directory]
python3 generate_logs.py
```

Expected output:

```
Sending [FILL: total] events to Splunk HEC...
[50/500] Events sent...
...
[500/500] Events sent...

--------------------------------------------------
Done.
  Total sent:    [FILL: total]
  Failed:        0
  Index:         [FILL: your-index-name]
```

The script runs in approximately 10–15 seconds.

**If you see errors:**

```
ERROR: HTTP 401 Unauthorized
```
Token mismatch. Verify you are using the UUID Token Value, not the token name.

```
ERROR: Connection refused — is Splunk running?
```
Start Splunk:
```bash
/Applications/splunk/bin/splunk start --accept-license
```
Wait 90 seconds and retry.

```
ERROR: HTTP 400 Bad Request
```
JSON payload issue or wrong index name. Verify `INDEX` and `SOURCETYPE` are set correctly.

---

## Step 4: Verify Events in Splunk

1. Open `http://localhost:8000` and log in
2. Click **Search & Reporting**
3. Switch search mode to **Verbose Mode** (top-right dropdown)
4. Run:

```spl
index=[FILL: your-index-name] | stats count by EventCode
```

Set time range to **Last 7 days**.

**Expected output:**

| EventCode | count |
|---|---|
| 4624 | ~[FILL: expected count] |
| 4625 | ~[FILL: expected count] |
| 4720 | [FILL: exact count] |
| 4722 | [FILL: exact count] |

> **Use Verbose Mode, not Smart Mode:**
> Smart Mode disables field data for stats searches. In Smart Mode, `stats count by EventCode` cannot find the `EventCode` field and returns a single row with count 1. Always switch to Verbose Mode before running any stats query.

> **Use Last 7 days, not Last 24 hours:**
> Some events are timestamped across the last 7 days. A 24-hour window will miss them.

[INSERT SCREENSHOT: Splunk stats count by EventCode showing expected distribution]

---

## Step 5: Spot-Check Brute Force Data

```spl
index=[FILL: your-index-name] EventCode=4625 | stats count by IpAddress | sort - count
```

Your designated brute-force IP (`[FILL: your-test-ip, e.g. 203.0.113.45]`) should appear at the top with the highest failure count. Other IPs should have significantly fewer failures.

If your brute-force IP is missing, re-run the script — events stack additively and the queries will still work correctly.

[INSERT SCREENSHOT: IpAddress failure count showing brute-force IP at top]

---

## Step 6: Spot-Check Admin Account Events

```spl
index=[FILL: your-index-name] (EventCode=4720 OR EventCode=4722) | table _time, EventCode, SubjectUserName, TargetUserName, Message
```

Expected: [FILL: e.g. 10] rows, with account creations (4720) and account enables (4722), each showing:
- `SubjectUserName` — who performed the action
- `TargetUserName` — which account was created or enabled
- `Message` — human-readable description

[INSERT SCREENSHOT: Admin account events table]

---

## Phase 3 Checkpoint

- [ ] Script ran without errors
- [ ] `stats count by EventCode` shows expected event distribution
- [ ] Brute-force IP appears at the top of the failure count
- [ ] 4720 and 4722 events show `SubjectUserName` and `TargetUserName` fields populated

---

## Data Design Notes

**Why [FILL: your brute-force count] failures from one IP?**
Your policy threshold is [FILL: e.g. 10/24h]. Using a count well above the threshold makes the detection unambiguous — the dashboard panel will clearly show a number exceeding the alert threshold.

**Why spread [FILL: historical count] events across 7 days?**
The trend chart in Phase 5 needs data across multiple days to show a meaningful line. Without historical data, the chart shows a single spike at the current time with nothing before it — which does not demonstrate continuous monitoring.

**Why capture both 4720 and 4722?**
A real admin account creation workflow produces both events: 4720 when the account object is created, 4722 when it is enabled (accounts are often created disabled and then explicitly enabled). Capturing both proves the entire account lifecycle is logged.

**Why standard Windows field names?**
Using `IpAddress`, `SubjectUserName`, and `TargetUserName` means your SPL queries transfer directly to a real enterprise Windows environment. No remapping is required when moving from this lab to a production SIEM.

---

## What Went Wrong: Phase 3 Dead Ends

### Dead End 1: json.dumps() Double-Serialization (HTTP 400)

**What happened:** The `build_event` function passed `json.dumps(event_fields)` as the `event` value in the HEC payload.

```python
# Broken
"event": json.dumps(event_fields)
```

**Result:** HTTP 400 Bad Request on every send. Zero events indexed.

**Why:** Splunk HEC expects the `event` field to be a JSON object (dict). Passing `json.dumps()` converts it to a string first, then the outer serialization in `send_event()` wraps it again — double-encoding. The fix:

```python
# Correct
"event": event_fields
```

**Lesson:** Never pre-serialize the `event` field. The outer `json.dumps()` in the send function handles full payload serialization.

---

### Dead End 2: Wrong SOURCETYPE Value

**What happened:** `SOURCETYPE` was set to `"WinEventLog"` instead of `"_json"`.

**Result:** Events sent with HTTP 200, but `stats count by EventCode` returned a single row with count 1.

**Why:** `WinEventLog` is a sourcetype for raw Windows event log text format. With this sourcetype, Splunk does not extract JSON fields. `EventCode`, `IpAddress`, and other fields do not exist as searchable fields.

**Lesson:** A successful HEC response only confirms the event was stored. It does not confirm fields were extracted. Always verify extraction separately using `stats count by EventCode` in Verbose Mode.

---

### Dead End 3: Smart Mode Hiding Field Extraction Failures

**What happened:** Splunk was left in Smart Mode (the default) while running `stats count by EventCode`.

**Result:** The query returned a single row with count 1 — even after the script sent all events correctly.

**Why:** Smart Mode disables field data for stats searches as a performance optimization. It makes it impossible to distinguish between a real field extraction failure and a search mode issue.

**Lesson:** Always use Verbose Mode when debugging field extraction. Switch back to Smart Mode only after confirming results are correct.

---

### Dead End 4: Stale Data From Failed Runs Polluting the Index

**What happened:** Multiple failed script runs (wrong token, wrong sourcetype, serialization bug) each injected partial or malformed events into the index.

**Result:** Queries returned inconsistent counts that didn't match expectations.

**Fix:** Stop Splunk, clean the index, restart, then re-run:

```bash
/Applications/splunk/bin/splunk stop
/Applications/splunk/bin/splunk clean eventdata -index [FILL: your-index-name]
/Applications/splunk/bin/splunk start
python3 generate_logs.py
```

> **Splunk must be fully stopped before cleaning.** Running the clean while Splunk is active silently does nothing.

**Lesson:** Clean the index before each test run. Stale events from failed runs will always produce confusing results.

---

## Next Phase

[→ Phase 4: SPL Queries](Phase-4-SPL-Queries.md)
