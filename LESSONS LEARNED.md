# Lessons Learned

In my compliance logging project, I created a Splunk pipeline that takes in simulated Windows security events and puts them on a dashboard I designed. This document captures every significant failure I hit, why it happened, and how I fixed it.

---

## Phase 1: Installation

### I Tried Three Ways to Install Splunk. None of Them Worked.

Before I found the right way, I spent some time on three paths that didn’t lead anywhere.

**Docker (ARM64 Mac):**  I thought the official Splunk Docker image would run smoothly on my M4 MacBook Air. It doesn't. Splunk has not published ARM64-compatible images to Docker Hub. All available manifests are `linux/amd64`. The platform directive fails immediately.

**Command-line download:** I tried pulling the installer directly:

```bash
wget https://www.splunk.com/page/download/latest -O splunk-latest-macos.dmg
```

This returned a 137 KB HTML redirect page instead of the 950 MB DMG. Splunk's download page is JavaScript-dependent and uses referrer-based gating. `curl -L` has the same problem.

**Homebrew:**

```bash
brew install splunk
```

No formula exists. Enterprise software typically requires direct downloads or license management, not package managers.

**What I learned:** Always check the official documentation before assuming a standard installation method works for Splunk on macOS, the standard `brew install` fails, leaving direct DMG downloads, command-line tarballs, or emulated Docker containers as the only reliable deployment paths.

---

## Phase 2: HEC Configuration

### The Setup Form Pre-fills the Wrong Port and Saves Without Error

HEC (HTTP Event Collector) is Splunk's mechanism for receiving log data over HTTP or HTTPS. When I opened the HEC Global Settings form, it auto-filled the port as `8000`—the exact same port the Splunk Web UI runs on. I saved it without noticing the collision.

The result: because HEC was incorrectly assigned to port `8000` (colliding with the web UI), every subsequent data-send attempt directed at the standard port `8088` returned a silent `connection refused` error. Splunk accepted the configuration layout without generating a system alert.

**Fix:** Always manually verify the port field is set to `8088` (or your intended custom HEC port) before saving. After saving, verify HEC is actively listening and bypassing self-signed SSL warnings by running:

```bash
curl -X POST https://localhost:8088/services/collector \
  -H "Authorization: Splunk YOUR_TOKEN_UUID_HERE" \
  -d '{"event": "test"}'
```

A working HEC returns `{"text":"Success","code":0}`. Anything else means the interface endpoint is misconfigured.

**What I learned:** A successful save in a UI does not mean the configuration is functionally valid. Test every technical integration point immediately after setup, rather than troubleshooting upstream after you have built dependencies on top of it.

---

### I Used the Token Name Instead of the Token Value and Got 401s

After setting up HEC, I created an authentication entry named `compliance-lab-token` and passed that string directly into my `Authorization` header. Every request consistently failed with an `HTTP 401 Unauthorized` response containing no detailed error context.

The issue stems from confusing the two distinct components of an HEC token definition:

- **Token Name:** A human-readable display label used purely for internal organization and management within the Splunk console (e.g., `compliance-lab-token`).

- **Token Value:** A system-generated, 36-character UUID string that acts as the actual API key required by the HTTP `Authorization` header.

```
Name:        compliance-lab-token       ← label only, never use this in code
Token Value: Splunk HEC_TOKEN_UUID_HERE  ← use this UUID
```

Correct header:
```bash
-H "Authorization: Splunk HEC_TOKEN_UUID_HERE"
```

**What I learned:** Copy the Token Value UUID immediately after creation. The name field is just a display label it has no functional role in authentication.

> Note: The UUID above is a placeholder example. Never commit real token values to public repositories.

---

## Phase 3: Sending Data

### HTTP 200 Does Not Mean Field Extraction Worked

I sent JSON-formatted events to the HEC endpoint and received a reassuring `{"text":"Success","code":0}` response. Assuming the data pipeline was fully functional, I moved straight to building dashboard panels. This was a mistake.

The underlying issue was a severe **source type mismatch**. The ingestion pipeline was instructing Splunk to treat the incoming payload as `WinEventLog:Security`, while the actual payload format was structured JSON. Because these two source types utilize completely contradictory parsing engines, the breakdown was catastrophic yet silent:

- `WinEventLog:Security` instructs Splunk's parsing pipeline to expect raw, space-delimited Windows Event Log text.

- The payload actually contained structured JSON key-value pairs.

Splunk successfully accepted and indexed the raw data packet (hence the HTTP 200), but the indexer failed to execute search-time field extractions. As a result, structural queries like `| stats count by EventCode` failed completely because `EventCode` was never extracted as an indexed field it was simply buried as unparsed text within the `_raw` blob.

**Fix:** Enforce the `_json` source type consistently across the entire pipeline to trigger Splunk's native JSON parsing engine.

**Fix:** Use `_json` consistently on both ends.

1. **In the Data Generator Script:** Ensure the metadata payload specifies:
```python
SOURCETYPE = "_json"
```

2. **In the Splunk HEC Input Configuration:** Ensure the default Source Type is explicitly set to `_json`.

After re-sending the data, verify that search-time field extraction is functioning properly by executing a query in **Verbose Mode** (Smart Mode can sometimes hide unreferenced fields):

```
index=compliance_demo | fields *
```

**Verification Criteria:** Fields such as `EventCode`, `IpAddress`, and `SubjectUserName` must appear natively in the Left Indexer Panel and as distinct columns in your statistics table, rather than existing solely as raw text strings inside `_raw`.

**What I Learned:** An HTTP 200 or HEC "Success" message only confirms that the transport layer successfully handed off the bytes to the indexer. It offers no guarantee regarding data utility or parsing validity. Always validate your schema and field extractions before building upstream analytics or dashboards.

---

## Phase 4: Dashboard Build

### Empty Panels Because Field Names Didn't Match

I constructed a classic dashboard using Simple XML, defining panels based on assumed field names that mirrored my data generator's internal variable names (e.g., `src_ip`, `user`). Although a baseline search proved that events were actively landing in the index, the dashboard panels rendered entirely empty.

The failure stemmed from a mismatch between my assumed schema and the raw JSON keys. Splunk automatically extracts JSON keys exactly as they are formatted in the incoming payload. Because field names in Splunk are strictly case-sensitive, my XML source code was querying non-existent fields:

``` 
Assumed XML Fields (Failed):  src_ip        user
Actual JSON Keys (Extracted):  IpAddress     SubjectUserName
```

Because Splunk treats unextracted fields as valid but empty, the search engine returned zero rows silently, without throwing syntax errors or warnings.

**Fix:** Never write dashboard XML or complex SPL queries based on assumptions. Always run an exploratory discovery search in **Verbose Mode** first to audit the live schema:

```
index=compliance_demo | fields *
```

Expand the event viewer or audit the Left Indexer Panel to copy the exact case-sensitive field strings. A single character casing mismatch (e.g., using `ipaddress` instead of `IpAddress`) will completely break a panel.

**What I Learned:** The visual UI layout is entirely dependent on the precision of the underlying data contract. Always validate your extracted field schema at the search layer before hardcoding fields into dashboard source code.

---

### Timestamps Displayed as Raw Unix Integers

The Admin Activity Audit panel showed timestamps like `1747267200` instead of `2025-05-14 10:30:00`. The data was correct—the display was not.

Splunk stores time internally as a Unix integer. The `table` command renders `_time` as that raw number by default.

**Fix:** **Fix:** To make sure compliance format is just right, use the `strftime` (string-format time) function inside an `eval` statement. This way, I can clearly tell the function how to format the integer before it gets sent to the final visualization table.

```spl
| eval Timestamp=strftime(_time, "%Y-%m-%d %H:%M:%S")
| table Timestamp, event_type, SubjectUserName, IpAddress
```

Common Formatting Codes Reference:

- `%Y-%m-%d`: Outputs Year-Month-Day (e.g., 2025-05-14)
- `%H:%M:%S`: Outputs 24-hour clock Time (e.g., 10:30:00)

**What I Learned:** don’t expect raw backend metric fields to look good when you present them. To make sure your dashboard panel is ready for audits, I need to transform those mathematical system epochs into clear, easy-to-read timestamps.

---

### Changing `<dashboard>` to `<form>` Added Unwanted Search Boxes

I wanted to add a time picker to the dashboard, so I changed the XML root element from `<dashboard>` to `<form>`. Splunk immediately rendered a search bar labeled "No title" above every panel.

The `<form>` element is designed for parameterized, user-driven dashboards. It automatically injects search input UI elements. For a compliance dashboard with fixed time ranges per panel, it's the wrong element.

**Fix:** Keep the root as `<dashboard>`. Set time ranges inside each panel's search block:

```xml
<search>
  <earliest>-7d</earliest>
  <latest>now</latest>
  <query>index=compliance_demo | stats count by EventCode</query>
</search>
```

---

### Brute Force Panel Was Empty Because the Time Scope Was Too Narrow

My brute force panel was scoped to `-24h` and showed no data, even though I knew events existed. The `generate_logs.py` script randomizes brute force event timestamps across the last 7 days not just the last 24 hours. A 24-hour window missed most of the dataset.

**Fix:** Set the panel time range to `-7d` to match the script's actual data distribution.

---

## Debugging Reference

### Smart Mode Silently Returns Wrong Results

I found out that Splunk defaults to **Smart Mode** to toggle field discovery based on search characteristics. When running transforming commands (like `stats`, `chart`, or `timechart`), Smart Mode will tell the engine to pause extracting fields that aren’t really needed, which can help save CPU power.

**The Fix:** When troubleshooting data onboarding or auditing raw field schemas, switch the search execution toggle to Verbose Mode. This mode forces the indexing layer to extract and display every discoverable field variable, even if it’s not explicitly named in the SPL query string.

### Splunk's Three Ports Serve Three Different Purposes

| Port   | Purpose                              | Security & Traffic Profile                                                       |
| ------ | ------------------------------------ | -------------------------------------------------------------------------------- |
| `8000` | Splunk Web UI                        | Unencrypted or SSL-encrypted browser access for analysts and admins.             |
| `8088` | HEC data input (where logs are sent) | Dedicated data ingestion pipeline for API-driven tokens and automated logging.   |
| `8089` | **Management API (splunkd)**         | Secure deployment server communication, CLI commands, and cluster orchestration. |

**What I Learned:**

1. Choose your search mode based on your objective: use Smart/Fast Mode for high-speed dashboard rendering, and Verbose Mode for pipeline validation and schema auditing.

2. Network port discipline is mandatory. Keep track of how background browser auto-fill components interact with enterprise administrative interfaces.

---

## Quick Reference

| **What Broke**                                               | **True Root Cause**                                                                                                                                   | **Engineering Fix / Workaround**                                                                                                                          |
| ------------------------------------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Splunk deployment fails** on Apple Silicon                 | Mainline `brew install` lacks a core formula; official Docker images lack native `linux/arm64` manifests.                                             | Use the native macOS **command-line Tarball (`.tgz`)**, direct DMG, or run the Docker image with the `--platform linux/amd64` flag via Rosetta emulation. |
| **HEC endpoint not listening** on port `8088`                | Local browser auto-fill templates aggressively overwrite the HEC port field with the cached Web UI port (`8000`).                                     | Manually audit the field to ensure it is set to `8088`. Validate live connectivity using `curl -k`.                                                       |
| **HTTP 401 Unauthorized** on every HEC request               | The human-readable Token Name string was passed into the `Authorization` header instead of the secret key.                                            | Extract and pass the system-generated **Token Value (UUID)** into the header string: `Authorization: Splunk <UUID>`.                                      |
| **Fields missing** after successful data ingestion           | A source type mismatch occurred. Structured JSON data was indexed using an unstructured parser ruleset (e.g., `WinEventLog:Security`).                | Explicitly enforce the `_json` source type within both the data generator payload metadata and the Splunk HEC input settings.                             |
| **Dashboard panels render completely empty**                 | Splunk field extractions are strictly case-sensitive. The XML source code queried assumed variables (`src_ip`) instead of live keys (`IpAddress`).    | Execute an exploratory query (`                                                                                                                           |
| **Timestamps render as raw** 10-digit integers               | Custom epoch values parsed out of JSON strings are treated as plain numbers by the presentation layer.                                                | Apply the `strftime` function within an `eval` statement to format the epoch string (e.g., `"%Y-%m-%d %H:%M:%S"`) prior to the `table` command.           |
| **Unwanted inputs/search boxes** added to dashboard          | Altering the root XML tag to `<form>` forces Splunk to automatically generate a blank, global `<fieldset>` user-input header block.                   | Revert the root element to `<dashboard>` for hardcoded data horizons, or explicitly build out the `<fieldset>` inputs if a global controller is required. |
| **Security anomaly panels** return zero results              | The dashboard panel query window (e.g., `-24h`) was tighter than the operational distribution window of the underlying log generator script.          | Shift the panel's time boundary to `-7d` to align the SIEM's query window with the simulation data's temporal distribution scale.                         |
| **Field sidebar disappears** during search queries           | **Smart Mode** optimizes system performance by automatically suppressing the discovery of unreferenced fields when transforming commands are present. | Toggle the search execution drop-down to **Verbose Mode** when actively debugging data ingestion pipelines or validating fresh schemas.                   |


