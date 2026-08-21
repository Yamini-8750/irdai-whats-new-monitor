# IRDAI "What's New" Monitor (n8n)

A daily automation that watches the IRDAI regulator's "What's New" page, detects newly published documents, and emails the research team a digest — so no regulatory update is ever missed.

> **Impact:** Eliminated daily manual checking of the IRDAI portal and closed a real gap: regulatory updates were easy to miss given the volume and frequency of publications, and missing one directly affects the accuracy of insurance ratings and research. The monitor runs **every day at 9:00 AM IST** and reliably surfaces new circulars, consultation papers, notifications, and orders.

<!-- Add a short Loom showing a run + the resulting email digest -->
**▶️ Demo:** [Link](https://drive.google.com/file/d/1sUwoqOm8QZvEaBsdgSLq1gdsy-dgSmmH/view?usp=drive_link)


---

## The problem

The IRDAI website publishes circulars, consultation papers, press releases, and regulatory orders in a "What's New" section. Standard change-detection tooling returned a `403 Access Denied` on the site, making off-the-shelf scraping impossible — and manual daily checks weren't reliable.

## The insight

On investigation, the "What's New" page loads its items from a JavaScript array (`DLFileEntryArray`) embedded in the page HTML. An **n8n HTTP Request node with browser-like headers** can fetch the page where change-detection tools were blocked — and that embedded array is clean, structured JSON (date, title, category, document ID) for every publication. That made a lightweight, no-scraping-service solution possible entirely inside an existing n8n + Google Sheets stack.

## How it works

```
Schedule (9:00 AM IST daily)
        │
        ▼
HTTP Request  → fetch IRDAI "What's New" with browser headers
        │
        ▼
Parse HTML    → extract DLFileEntryArray (date, title, category, URL, id)
        │
        ▼
Read Sheet    → load previously-seen document IDs
        │
        ▼
Diff          → keep only NEW items not seen before
        │
        ├─► Append new items to the "Seen" sheet (so they never re-alert)
        │
        ▼
Compose digest → one combined email (not one per item)
        │
        ▼
Send email    → daily digest to the research team
```

The Google Sheet acts as the system's **memory**: every alerted document ID is logged, and any item already in the sheet is skipped on the next run — which prevents duplicate alerts.

## Tech stack

`n8n` (self-hosted via Docker) · `JavaScript` (parsing + diff + digest nodes) · `Google Sheets API` (state store) · `Gmail / SMTP` (digest delivery) · `Docker`

## Workflow nodes

| # | Node | Role |
|---|---|---|
| 1 | Schedule Trigger | Fires daily at 9:00 AM IST |
| 2 | HTTP Request | Fetches the page with browser headers |
| 3 | Code (JS) | Extracts & parses `DLFileEntryArray` |
| 4 | Get rows | Reads previously-seen IDs from the sheet |
| 5 | Code (JS) | Diffs fetched vs seen → new items only |
| 6 | Append rows | Logs new items so they aren't re-alerted |
| 7 | Code (JS) | Builds one combined email digest |
| 8 | Send Email | Sends the digest to the research team |

## Running it

```bash
cd ~/your-monitor-folder
docker compose up -d          # start n8n + services
# n8n UI: http://localhost:5678
docker compose ps             # verify containers are Up
docker compose down           # stop everything
```

> ⚠️ Configure recipients, sheet ID, and SMTP credentials via n8n credentials / environment variables — **do not hard-code them** and do not commit them.

## Roadmap

- **Move to a cloud server** (e.g. always-on VM) so the workflow runs 24/7 independent of a local machine.
- **Dedicated sending domain** to keep digests out of spam.
- Optionally log updates to the main change-log sheet alongside insurer-website changes.

## What's in this repo

```
workflow.json         # exported n8n workflow (credentials removed)
parsing-node.js       # DLFileEntryArray extraction logic
docker-compose.yml    # service definition (no secrets)
.env.example          # names of required secrets
README.md
Recorded video N8N Server Working
```
