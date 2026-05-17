
- **Time spent:** ~45 minutes (including troubleshooting)
- **Goal:** Splunk is running locally, your index exists, and HEC is accepting events on port 8088.
- **Status:** Complete

← [Back to Index](README.md)

---

## Platform Note: Apple Silicon (M-Series Mac)

The standard guide assumes Docker. **On Apple Silicon, Docker does not work for this lab.** Splunk does not publish native ARM64 Docker images the `splunk/splunk` image targets `linux/amd64` only, and Docker Desktop cannot resolve an ARM64 manifest that doesn't exist.

**The correct path on Apple Silicon: native Mac installation via DMG.** It takes roughly 15 minutes, requires no container management, and reaches the same Splunk Web UI and HEC endpoints. Steps 5-8 are identical regardless of how Splunk was installed.

> **Note: If you're on Linux or an Intel Mac:**
> Docker works fine on those platforms. Skip to the *Docker Path* section at the bottom of this document, then rejoin at Step 5.

---

## What You're Building

A local Splunk Enterprise instance (trial license, downgrades to Free after 7 days) running natively on macOS. By the end of this phase:

- Splunk running at `http://localhost:8000`
- Admin credentials set by you during installation
- Index `[FILL: your-index-name]` created and enabled
- HEC endpoint active at `http://localhost:8088` with a UUID token

---

## Step 1: Verify Prerequisites

Open Terminal and run:

```bash
# Confirm Python 3 is available (needed for Phase 3)
python3 --version
```

Expected: Python 3.9 or higher.

```bash
# Confirm ports 8000 and 8088 are free
lsof -i:8000
lsof -i:8088
```

Expected: no output on both commands. If you see output, something else is using those ports stop it before continuing.

---

## Step 2: Download the DMG

Navigate to the official download page in your browser:

```
https://www.splunk.com/en_us/download/splunk-enterprise.html
```

- Select **macOS**
- Download the **Intel** version (the `.dmg` file, ~950 MB)

> **Note "Intel" on an M-series Mac:**
> The download page labels it Intel, but this DMG runs on M-series Macs via Rosetta 2. There is no ARM64-specific build. The Intel label refers to the binary architecture, not a compatibility warning.

The file will be named something like:

```
splunk-[VERSION]-[HASH]-darwin-intel.dmg
```

**Do not use `wget`, `curl`, or CDN shortcut URLs.** Splunk's download flow uses browser referrer headers — scripted downloads return an HTML redirect page, not the binary.

---

## Step 3: Mount and Install

```bash
open ~/Downloads/splunk-[VERSION]-[HASH]-darwin-intel.dmg
```

This opens a Finder window. Launch the **"Install Splunk [VERSION]"** wizard.

Follow the GUI installer:
- Destination: `/Applications`
- License: Accept the Enterprise Trial
- Installation type: Default

The progress bar takes approximately 2 minutes.

---

## Step 4: Start Splunk

After installation, click **"Start and Show Splunk"** in the wizard.

Splunk initializes and opens at `http://localhost:8000`. First boot takes ~90 seconds.

At the login screen, create your admin credentials:
- **Username:** `admin`
- **Password:** `[YOUR-PASSWORD]` - choose a strong password and save it somewhere secure

>
> **Security note:** Never commit your Splunk admin password to a public repository.

Log in and confirm you see the Splunk Home dashboard.

---

## Step 5: Create Your Index

**In the Splunk UI:**

1. Click **Settings** → **Indexes**
2. Click **New Index**
3. Fill in:
   - **Index Name:** `[FILL: your-index-name]` (e.g. `compliance_demo`)
   - **Max Size:** 500 MB (default is fine for this lab)
   - Leave everything else as-is
4. Click **Save**

Your index should appear in the list with status **Enabled**.

> **Why a dedicated index?**
> In a production SIEM, security events, network logs, and application logs live in separate indexes this speeds up searches and allows distinct retention policies. A dedicated index here means your SPL queries are precise and don't hit unrelated data.

index list showing your index as Enabled<img width="1914" height="533" alt="image" src="https://github.com/user-attachments/assets/6cf2ab58-b990-4ae2-b68d-d471d30636fb" />


---

## Step 6: Enable HTTP Event Collector (HEC)

HEC is the REST endpoint your Python script will use to send events. It is disabled by default.

**In the Splunk UI:**

1. Navigate to **Settings → Data Inputs**
2. Click **HTTP Event Collector**
3. Click **Global Settings** (top right)
4. Set **All Tokens** to **Enabled**
5. In the **HTTP Port Number** field, clear the pre-filled value and type `8088`

> Critical: Browser auto-fill managers frequently overwrite this configuration field with port 8000 (the Web UI port). You must manually ensure it is set to 8088; if saved as 8000, the HEC service will not open port 8088, causing data-delivery scripts targeting that port to fail with a clear TCP connection refused error.

6. Click **Save**

---

## Step 7: Create the HEC Token

Still on the HTTP Event Collector page:

1. Click **New Token**
2. Fill in:
   - **Name:** `[FILL: your-token-name]` (e.g. `compliance-lab-token`)
   - Click **Next**
3. On the Input Settings page:
   - **Source type:** `_json`
   - **Default Index:** `[FILL: your-index-name]`
   - Click **Review**, then **Submit**
4. Copy the **Token Value** it will be a UUID formatted as `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`

> **Token name vs. token value these are different things:**
> The Name is a human-readable label. The Token Value is the UUID Splunk generates. The `Authorization` header in your script and curl commands requires the UUID never the name. Using the name returns HTTP 401 with no explanation.

> **Source type must be `_json`:**
> This tells Splunk to parse incoming payloads as JSON and extract fields automatically. Without it, fields like `EventCode` and `IpAddress` are not extracted, and `stats count by EventCode` will return wrong results.

---

## Step 8: Verify HEC Is Working

Test the endpoint from Terminal before running any Python:

```bash
curl -k http://localhost:8088/services/collector/event \
  -H "Authorization: Splunk [YOUR-TOKEN-UUID]" \
  -d '{"index":"[FILL: your-index-name]","sourcetype":"_json","event":{"EventCode":4625,"Message":"Test event - HEC verification"}}'
```

Expected response:

```json
{"text":"Success","code":0}
```

If you get anything else: re-check the token UUID, confirm port 8088, and confirm the index name matches exactly.

> **Shell quoting note (zsh):**
> The curl command uses both double and single quotes. If a double quote appears inside the `-d` string, zsh will enter `dquote>` mode and freeze, waiting for a closing quote. Press `Ctrl+C` to exit and retype carefully.

**Verify the event landed:**

1. In the Splunk UI, click **Search & Reporting**
2. Run: `index=[FILL: your-index-name]`
3. Time range: **Last 15 minutes**
4. The test event should appear

If it does, Phase 1 is complete.

---

## Phase 1 Checkpoint

- [ ] Splunk running at `http://localhost:8000`, login works
- [ ] Index `[FILL: your-index-name]` exists and is Enabled
- [ ] HEC enabled on port 8088
- [ ] HEC token created with source type `_json`
- [ ] curl test returned `{"text":"Success","code":0}`
- [ ] Test event visible in Splunk search
- [ ] Splunk version noted: `[FILL: version]`

---

## Next Phase

[→ Phase 2: Policy Mapping](Phase-2-Policy-Mapping.md)
