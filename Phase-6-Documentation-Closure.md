---
title: Phase 6 - Documentation Closure
lab_id: "[FILL: Your Lab ID]"
phase: 6
type: lab-phase
status: complete
created: "[FILL: YYYY-MM-DD]"
tags: [splunk, nist, grc, lab]
---

**Time estimate:** 20–30 minutes
**Goal:** Finalize evidence, verify end-to-end traceability, and confirm the lab is portfolio-ready.

← [Back to Lab Index](README.md)

---

## What This Phase Does

This phase closes the loop between your policy documentation and the technical implementation. By the end:

- The dashboard screenshot is saved as evidence
- The traceability matrix is verified end-to-end
- You can walk the full audit chain from memory for both AC-7 and AU-9
- The lab is marked complete and ready for your portfolio or GitHub

---

## Step 1 — Capture the Portfolio Screenshot

Before finalizing, take a screenshot of the live dashboard that shows all four panels.

**Requirements:**
- All four panels visible in a single frame
- Dashboard title clearly visible
- Panel 1 (single value) showing a red or orange alert state
- Panel 2 (table) showing your brute-force IP
- Panel 3 (table) showing at least 2–3 rows of account activity
- Panel 4 (line chart) showing both Success and Failure lines over 7 days

**How to take it (macOS):**

Press `Cmd+Shift+4` and drag to capture the dashboard.

Save the file as:

```
screenshots/nist-compliance-dashboard.png
```

This is your primary portfolio evidence. It belongs in the `screenshots/` folder of your repository.

---

## Step 2 — Update Your Policy Document

If you have a prior policy document (e.g. from an ISMS lab), open it and update the technical validation section.

Add the following to your policy's evidence section:

```markdown
## Technical Validation

**Screenshot Reference:**

![NIST Compliance Dashboard](../screenshots/nist-compliance-dashboard.png)

*Screenshot taken [FILL: DATE]. Dashboard shows all four panels:
Failed Logins (AC-7), Brute Force Source (AC-7), Admin Activity Audit (AU-9),
and Authentication Trends (DE.CM-01). Single value panel shows [FILL: COUNT]
failures in the last 24 hours — above the [FILL: THRESHOLD]-failure alert
threshold defined in [FILL: policy section].*
```

Also update your policy document's status from `draft` to `active` if applicable.

> **Note:** If you do not have a prior policy document, this step can be skipped. The traceability matrix below serves as your primary evidence artifact.

---

## Step 3 — Walkable Audit Chain

Practice walking this chain from memory before calling the lab done. You should be able to do this without looking at any notes during an interview.

**For AC-7 (Brute Force Detection):**

```
Policy Mandate          Framework Mapping       Ingestion Contract
[FILL: policy section]  NIST SP 800-53 AC-7     Windows Event ID 4625
[FILL: threshold/24h]   CSF 2.0 DE.AE-02        Structured JSON via HEC

         |
         v

Presentation Layer      Analytics Pipeline      Parsing Engine
Dashboard Panel 2       SPL Filter Constraint   Automatic Key Extraction
"Brute Force Sources"   where failures >= N     via _json Sourcetype
```

**For AU-9 (Admin Account Monitoring):**

```
Policy Mandate          Framework Mapping       Ingestion Contract
[FILL: policy section]  NIST SP 800-53 AU-9     Windows Event IDs 4720, 4722
Any account creation    CSF 2.0 DE.CM-03        Structured JSON via HEC
triggers investigation

         |
         v

Presentation Layer      Analytics Pipeline      Parsing Engine
Dashboard Panel 3       eval + table + sort     Automatic Key Extraction
"Admin Activity Audit"  SubjectUserName,        via _json Sourcetype
                        TargetUserName
```

If you can walk both chains without reading this document, the lab is done.

---

## Traceability Matrix

This is the complete audit-ready mapping from policy to technical proof.

| Control ID | Regulatory Framework | Target Event ID | System Meaning | SPL Logic | Dashboard Panel | Policy Reference |
|---|---|---|---|---|---|---|
| AC-7 | NIST SP 800-53 | 4625 | Failed Logon | `stats count AS failed_logins` | Panel 1: Single Value | [FILL: policy §] |
| AC-7 | NIST SP 800-53 | 4625 | Failed Logon by Source | `stats count BY IpAddress \| where failures >= N` | Panel 2: Brute Force Table | [FILL: policy §] |
| AU-9 | NIST SP 800-53 | 4720 | Account Object Created | `table Timestamp, event_type, SubjectUserName, TargetUserName` | Panel 3: Admin Activity | [FILL: policy §] |
| AU-9 | NIST SP 800-53 | 4722 | Account Control Enabled | `table Timestamp, event_type, SubjectUserName, TargetUserName` | Panel 3: Admin Activity | [FILL: policy §] |
| DE.CM-01 | NIST CSF 2.0 | 4624, 4625 | Authentication Trend | `timechart span=1d count BY auth_result` | Panel 4: 7-Day Line Chart | [FILL: policy §] |
| DE.AE-02 | NIST CSF 2.0 | 4624, 4625 | Success/Failure Delta | `timechart span=1d count BY auth_result` | Panel 4: 7-Day Line Chart | [FILL: policy §] |

---

## Phase 6 Checkpoint

- [ ] Dashboard screenshot saved to `screenshots/nist-compliance-dashboard.png`
- [ ] Policy document updated with screenshot and date (if applicable)
- [ ] Traceability matrix verified — all rows accurate
- [ ] Audit chain walkable from memory for AC-7
- [ ] Audit chain walkable from memory for AU-9
- [ ] Lab README updated: `status: complete`, `portfolio_ready: true`
- [ ] Repository pushed to GitHub

---

## Lab Complete

When all checkboxes above are checked, this lab is portfolio-ready.

**Next step:** Push the repository to GitHub. If you have not initialized git in your lab folder yet:

```bash
cd /path/to/your/lab-folder
git init
git add .
git commit -m "Initial commit: Lab-[FILL: ID] complete"
git remote add origin https://github.com/[FILL: your-username]/[FILL: your-repo-name].git
git push -u origin main
```
