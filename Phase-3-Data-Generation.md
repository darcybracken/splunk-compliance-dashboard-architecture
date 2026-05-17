# Phase 3 Data Generation

- **Time estimate:** 20–30 minutes
- **Goal:** Run `generate_logs.py` and confirm all events appear in your Splunk index.

← [Back to Index](README.md)

---

## What the Script Does

`generate_logs.py` generates synthetic Windows Event Log events and sends them directly to Splunk via HEC. There are no intermediate log files — the script POSTs JSON directly to `http://localhost:8088`.

**Event breakdown:**

| Event Type | Event ID | Purpose | 
|---|---|---|
| Brute force failures (single IP) | 4625 | Triggers AC-7 alert one IP exceeds threshold |
| Normal failed logins (multiple IPs) | 4625 | Background noise spreads failures across many IPs |
| Admin account creations | 4720 | AU-9 account creation audit trail |
| Admin account enables | 4722 | AU-9 account enable audit trail |
| Privileged logins | 4624 | Admin logins mixed into success data |
| Normal successful logins | 4624 | DE.CM-01 baseline data |
| Historical events (7-day spread) | 4624, 4625 | Fills the trend chart with data across 7 days |
| **Total** | |

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
> - `SPLUNK_HOST` must use port `8088` not `8089` (management API) or `8000` (web UI)
> - `HEC_TOKEN` must be the UUID token value, not the token name
> - `INDEX` must match exactly it is case-sensitive
> - `SOURCETYPE` must be `_json` without this, Splunk accepts events but does not extract fields

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
310
182

| EventCode | count |
|---|---|
| 4624 | 310 |
| 4625 | 182 |
| 4720 | 5 |
| 4722 | 5 |

> **Use Verbose Mode, not Smart Mode:**
> Smart Mode disables field data for stats searches. In Smart Mode, `stats count by EventCode` cannot find the `EventCode` field and returns a single row with count 1. Always switch to Verbose Mode before running any stats query.

> **Use Last 7 days, not Last 24 hours:**
> Some events are timestamped across the last 7 days. A 24-hour window will miss them.

Splunk stats count by EventCode showing expected distribution
<img width="1918" height="507" alt="image" src="https://github.com/user-attachments/assets/b48ee04b-3f1c-427a-8538-f4056439c436" />


---

## Step 5: Spot-Check Brute Force Data

```spl
index=[FILL: your-index-name] EventCode=4625 | stats count by IpAddress | sort - count
```

Your designated brute-force IP (`[FILL: your-test-ip, e.g. 203.0.113.45]`) should appear at the top with the highest failure count. Other IPs should have significantly fewer failures.

If your brute-force IP is missing, re-run the script events stack additively and the queries will still work correctly.


---

## Step 6: Spot-Check Admin Account Events

```spl
index=[FILL: your-index-name] (EventCode=4720 OR EventCode=4722) | table _time, EventCode, SubjectUserName, TargetUserName, Message
```

Expected: With account creations (4720) and account enables (4722), each showing:
- `SubjectUserName` who performed the action
- `TargetUserName` which account was created or enabled
- `Message` human-readable description

---

## Phase 3 Checkpoint

- [ ] Script ran without errors
- [ ] `stats count by EventCode` shows expected event distribution
- [ ] Brute-force IP appears at the top of the failure count
- [ ] 4720 and 4722 events show `SubjectUserName` and `TargetUserName` fields populated

---

## Next Phase

[→ Phase 4: SPL Queries](Phase-4-SPL-Queries.md)
