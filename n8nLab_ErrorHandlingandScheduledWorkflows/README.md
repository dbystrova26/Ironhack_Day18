# Lab: Error Handling and Scheduled Workflows in n8n

## Overview

This lab demonstrates how to build production-ready workflows in n8n by adding error handling, retry logic, scheduling, and idempotency checks. Two workflows were created: a **Daily Data Fetcher** with error handling, and a **Daily Summary Generator** that runs on a schedule.

---

## Repository Structure

```
├── README.md               # This file — full walkthrough
├── lab_summary.md          # One-paragraph conceptual summary
└── screenshots/
└── json_files/             # n8n workflow schemas in json format


---

## Workflow 1: Daily Data Fetcher (Error Handling)

### What it does
Fetches data from the GitHub API. If the request fails, it retries automatically and routes to an error handler that formats a notification.

### How it was built

**Step 1 — Base workflow**
- Added a **Manual Trigger** node as the entry point for testing
- Added an **HTTP Request** node configured with:
  - Method: `GET`
  - URL: `https://api.github.com/users/octocat`
- Added an **Edit Fields** node to process the response data

**Step 2 — Error Trigger**
- Added an **Error Trigger** node to the canvas (separate from the main flow)
- In n8n, this node works at the workflow level — no manual arrow connection needed
- It automatically receives error details from any failing node in the workflow
- Output includes: `error.message`, `error.node.name`, `execution.id`, `workflow.name`

**Step 3 — Error Notification**
- Added an **Edit Fields** node after the Error Trigger
- Configured three fields to format the error:
  - `Error Message` → `{{ $json.error.message }}`
  - `Node Name` → `{{ $json['error']['node']['name'] }}`
  - `Timestamp` → `{{ $now }}`
- Added a **No Operation** node as a placeholder (in production: replace with Email/Discord/Webhook node)

**Step 4 — Retry Logic**
- Clicked the **HTTP Request** node → **Settings** tab
- Enabled **Retry On Fail**
- Set **Max Tries:** `3`
- Set **Wait Between Tries:** `5000` ms (5 seconds)
- Why: retries handle transient failures (timeouts, rate limits, 500 errors) without immediately triggering the error handler

**Step 5 — Testing**
- Changed the HTTP Request URL to `https://invalid-url-that-does-not-exist.com` to trigger a failure
- Verified the Error Trigger fired and the error was formatted correctly
- Verified retry attempts appeared in execution history
- Restored the URL to `https://api.github.com/users/octocat` and confirmed successful execution

### Final structure
```
Manual Trigger → HTTP Request (retry x3) → Edit Fields

Error Trigger → Edit Fields (format error) → No Operation
```

---

## Workflow 2: Daily Summary Generator (Scheduled + Idempotent)

### What it does
Runs automatically every day at 9 AM, fetches GitHub user data, checks if today's data has already been processed, and skips if it has — preventing duplicates.

### How it was built

**Step 6 — Schedule Trigger**
- Created a new workflow named **"Daily Summary Generator"**
- Added a **Schedule Trigger** node
- Configured:
  - Trigger Interval: `Days`
  - Days Between Triggers: `1`
  - Trigger at Hour: `9am`
  - Trigger at Minute: `0`
  - Timezone: `Europe/Berlin (UTC+02:00)`
- Why 9 AM: represents a realistic morning data pipeline that would run before business hours start

**Step 7 — Workflow Logic**
- Added **HTTP Request** node:
  - Method: `GET`
  - URL: `https://api.github.com/users/octocat`
- Added **Edit Fields** node to process and enrich the data:
  - `Summary Date` → `{{ $now.toISO().split('T')[0] }}` (e.g. `2026-05-14`)
  - `Username` → `{{ $json.login }}`
  - `Followers` → `{{ $json.followers }}`
- The `Summary Date` field becomes the unique key for idempotency

**Step 8 — Idempotency Check**
- Added an **IF** node after Edit Fields
- Configured condition:
  - Left value: `{{ $json['Summary Date'] }}`
  - Operator: `is equal to`
  - Right value: `{{ $now.toISO().split('T')[0] }}`
- **True branch** → **No Operation** ("already ran today — skip")
- **False branch** → **No Operation** ("new data — would save in production")
- Why this works: the workflow always compares today's date to the date it already tagged the data with — on the first run they match (True = processed), on re-runs the same day they still match (True = skip duplicate)

**Step 9 — Testing**
- Clicked **Execute Workflow** to run immediately without waiting for the schedule
- Verified the IF node routed to the **True branch** (idempotency working)
- Ran it a second time — confirmed it still routed to True (no duplicate processing)

**Step 10 — Error Handling Added to Scheduled Workflow**
- Added **Error Trigger** node to the canvas
- Added **Edit Fields** node after it with same error formatting:
  - `Error Message` → `{{ $json.error.message }}`
  - `Node Name` → `{{ $json['error']['node']['name'] }}`
  - `Timestamp` → `{{ $now }}`
- Added **No Operation** as placeholder for notification
- Enabled **Retry On Fail** on HTTP Request node (Max: 3, Wait: 5000ms)
- Activated the workflow using the **Inactive → Active** toggle

### Final structure
```
Schedule Trigger → HTTP Request (retry x3) → Edit Fields → IF
                                                         ├─ true → No Operation (skip)
                                                         └─ false → No Operation (save)

Error Trigger → Edit Fields (format error) → No Operation
```

---

## Key Concepts Learned

| Concept | Implementation |
|---|---|
| Error handling | Error Trigger node catches all workflow failures automatically |
| Retry logic | Retry On Fail setting on HTTP Request (3 retries, 5s delay) |
| Scheduling | Schedule Trigger node, daily at 9 AM Europe/Berlin |
| Idempotency | Date-based IF check prevents duplicate processing |
| Error notification | Edit Fields formats error details for downstream alerting |

---

## How to Import These Workflows

1. Open your n8n instance
2. Click **"New Workflow"**
3. Click the **"..."** menu → **"Import from JSON"**
4. Paste the exported workflow JSON
5. Review credentials and node configurations
6. Click **Save** and toggle **Active**
