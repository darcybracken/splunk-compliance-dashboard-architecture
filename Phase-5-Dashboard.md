
# Phase 5 Dashboard

- **Time estimate:** 15–20 minutes
- **Goal:** Import the Splunk dashboard XML and confirm all four panels render with live data.

← [Back to Index](README.md)

---

## Dashboard Overview

The dashboard has four panels, each proving a specific NIST control:

| Panel | Type | Control Proved | Time Range |
|---|---|---|---|
| Failed Logins Last 24 Hours | Single value (color-coded by threshold) | AC-7 | Last 24 hours |
| Brute Force Sources | Table | AC-7 | Last 7 days |
| Admin Activity Audit | Table | AU-9 | Last 7 days |
| Authentication Trends | Line chart | DE.CM-01 | Last 7 days |

---

## Step 1 Create a New Dashboard

1. In the Splunk UI, click **Search & Reporting**
2. Click **Dashboards** in the top navigation
3. Click **Create New Dashboard**
4. Fill in:
   - **Title:** `[FILL: your dashboard title, e.g. NIST Compliance Dashboard]`
   - **ID:** auto-fills from the title
   - **Description:** `[FILL: brief description, e.g. AC-7 and AU-9 control validation]`
   - **Permissions:** Private
5. Select **Classic Dashboard** (not Dashboard Studio)
6. Click **Create Dashboard**

> **Select Classic Dashboard, not Dashboard Studio:**
> Dashboard Studio uses a JSON-based format. The XML in this phase is written for Classic Dashboards only and will not work in Dashboard Studio. If you select Dashboard Studio by mistake, delete the dashboard and start over.

---

## Step 2 Switch to Source View

1. In the dashboard editor, click the **Source** button (top right, `< >` icon)
2. Select all existing content and delete it
3. Paste the full XML from Step 3

---

## Step 3 — Paste the Dashboard XML

Before pasting, replace all `[FILL: your-index-name]` placeholders with your actual index name.

```xml
<dashboard version="1.1">
  <label>[FILL: Your Dashboard Title]</label>
  <description>[FILL: Your dashboard description]</description>

  <row>
    <panel>
      <title>Failed Logins - Last 24 hours (AC-7)</title>
      <single>
        <search>
          <query>index=[FILL: your-index-name] EventCode=4625 | stats count AS failed_logins</query>
          <earliest>-24h</earliest>
          <latest>now</latest>
        </search>
        <option name="drilldown">none</option>
        <option name="colorBy">value</option>
        <option name="colorMode">block</option>
        <option name="rangeColors">["0x53A051","0xF8BE34","0xF1813F","0xDC4E41"]</option>
        <option name="rangeValues">[0,5,[FILL: your-threshold]]</option>
        <option name="useColors">1</option>
        <option name="numberPrecision">0</option>
        <option name="unit">failures</option>
      </single>
    </panel>

    <panel>
      <title>Brute Force Sources - IPs with [FILL: threshold]+ Failures (AC-7)</title>
      <table>
        <search>
          <query>index=[FILL: your-index-name] EventCode=4625 | stats count AS failures BY IpAddress | sort - failures | where failures >= [FILL: your-threshold] | rename IpAddress AS "Source IP", failures AS "Failed Attempts"</query>
          <earliest>-7d</earliest>
          <latest>now</latest>
        </search>
        <option name="drilldown">none</option>
        <option name="count">10</option>
      </table>
    </panel>
  </row>

  <row>
    <panel>
      <title>Admin Activity Audit - Account Creation and Modification (AU-9)</title>
      <table>
        <search>
          <query>index=[FILL: your-index-name] (EventCode=4720 OR EventCode=4722) | eval event_type=case(EventCode="4720", "Account Created", EventCode="4722", "Account Enabled", true(), "Unknown") | eval Timestamp=strftime(_time, "%Y-%m-%d %H:%M:%S") | table Timestamp, event_type, SubjectUserName, TargetUserName, Message | sort - Timestamp | rename event_type AS "Event Type", SubjectUserName AS "Performed By", TargetUserName AS "Target Account", Message AS "Details"</query>
          <earliest>-7d</earliest>
          <latest>now</latest>
        </search>
        <option name="drilldown">none</option>
        <option name="count">20</option>
      </table>
    </panel>
  </row>

  <row>
    <panel>
      <title>Authentication Trends - 7 Days (DE.CM-01)</title>
      <chart>
        <search>
          <query>index=[FILL: your-index-name] (EventCode=4624 OR EventCode=4625) | eval auth_result=case(EventCode="4624", "Success", EventCode="4625", "Failure", true(), "Unknown") | timechart span=1d count BY auth_result</query>
          <earliest>-7d</earliest>
          <latest>now</latest>
        </search>
        <option name="charting.chart">line</option>
        <option name="charting.chart.nullValueMode">connect</option>
        <option name="charting.legend.placement">bottom</option>
        <option name="charting.axisTitleX.text">Date</option>
        <option name="charting.axisTitleY.text">Event Count</option>
        <option name="drilldown">none</option>
      </chart>
    </panel>
  </row>

</dashboard>
```

---

## Step 4 Save and Preview

1. Click **Save**
2. Click **Back** to exit edit mode
3. The dashboard renders immediately with all four panels populated

**Expected result:**
- Panel 1: A large number displayed in color (red/orange means above threshold)
- Panel 2: A table with your brute-force IP at the top
- Panel 3: A table with account creation and enable events, readable timestamps
- Panel 4: A line chart with two lines (Success and Failure) over 7 days

Complete dashboard with all four panels visible
<img width="1080" height="1033" alt="image" src="https://github.com/user-attachments/assets/889af112-4807-4115-b8a0-957e66907b3d" />


---

## Troubleshooting

**Panel 1 is green (not red/orange):**
The count is below your threshold, which means "Last 24 hours" is not catching your failure events. This happens if you ran `generate_logs.py` more than 24 hours ago re-run the script to refresh events.

Color thresholds in the XML are:
- 0 to 4: green (`0x53A051`)
- 5 to 9: yellow (`0xF8BE34`)
- 10 to threshold-1: orange (`0xF1813F`)
- threshold+: red (`0xDC4E41`)

Adjust `rangeValues` in the XML to match your specific policy threshold.

**Panel 2 is empty:**
The `where failures >= [threshold]` filter is removing all results. Run Query 2 from Phase 4 directly in Search to confirm your brute-force IP is in the data. If not, re-run the script.

**Panel 3 shows 0 rows:**
Time range issue. Confirm events exist: `index=[FILL: your-index-name] (EventCode=4720 OR EventCode=4722)` with "All time."

**Panel 3 Timestamp shows large integers:**
The `strftime` eval is missing or malformed. Confirm `| eval Timestamp=strftime(_time, "%Y-%m-%d %H:%M:%S")` appears before the `table` command in the XML.

**Panel 4 shows only one point on the chart:**
Historical events did not spread correctly. Check: `index=[FILL: your-index-name] earliest=-7d@d | stats count by date_mday`. If you see only one date, re-run `generate_logs.py`.

---

## Phase 5 Checkpoint

- [ ] Dashboard created and saved in Splunk
- [ ] Panel 1 shows a number in color (red/orange means above threshold)
- [ ] Panel 2 shows your brute-force IP with a high failure count
- [ ] Panel 3 shows account creation and enable events with readable timestamps
- [ ] Panel 4 shows a line chart with both Success and Failure lines over 7 days

---

## Next Phase

[→ Phase 6: Documentation Closure](Phase-6-Documentation-Closure.md)
