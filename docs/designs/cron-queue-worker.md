# Daily Briefing System
## 5 AM Cron & Queue Worker — System Design

**Status:** Design complete — ready for implementation planning
**Last Updated:** April 2026
**Author:** Jack

> **Relationship to the PRD:** this design supersedes the simpler serial-processing description in [PRD §14.4](../PRD.md#144-5-am-cron--queue-worker-data-flow) and elaborates on [PRD §11](../PRD.md#11-parsed-document-store--task-queue). Instead of one worker processing its queue serially, this design fans out parallel Cloud Tasks per user per data source via Cloud Tasks queues, and guarantees a 6 AM briefing-generator task is scheduled regardless of whether the data tasks succeed.

---

## Overview

The 5 AM cron is the data ingestion engine for the Daily Briefing System. It fires once per day, fans out parallel data-source tasks for every active user via Cloud Tasks, and guarantees that a 6 AM briefing-generator task is scheduled for each user regardless of whether the data tasks succeed. The briefing generator works with whatever parsed documents are ready at 6 AM — partial data produces a degraded but still useful briefing.

---

## Architecture Summary

```
Cloud Scheduler — 5:00 AM
└── inbox-poller (Cloud Run job)
    └── For each active user:
        ├── Cloud Task → email-parse worker
        ├── Cloud Task → calendar-parse worker
        ├── Cloud Task → tasks-parse worker
        └── Cloud Task → briefing-generator (scheduled delivery: 6:00 AM)
```

All four tasks are enqueued in parallel per user. The data-source tasks (email, calendar, tasks) execute immediately and in parallel. The briefing-generator task sits in the Cloud Tasks queue with a scheduled delivery time of 6:00 AM — it fires whether or not the data tasks have completed successfully.

---

## Components

### 1. Cloud Scheduler — 5:00 AM Trigger

A single Cloud Scheduler job fires at 5:00 AM daily and makes an authenticated HTTP POST to the inbox-poller Cloud Run service.

| Field | Value |
|---|---|
| Schedule | `0 5 * * *` (5:00 AM daily) |
| Target | `POST /run` on inbox-poller Cloud Run service |
| Auth | OIDC token — Cloud Scheduler service account |
| Timeout | 10 minutes (fan-out only — actual work happens in Cloud Tasks) |

The inbox-poller's only job at this stage is fan-out: query active users and enqueue tasks. It does no data fetching itself.

---

### 2. inbox-poller (Cloud Run Job)

**Responsibility:** Query active users from Firestore, enqueue four Cloud Tasks per user, record the run in Firestore.

**Logic:**

1. Query `users` collection — fetch all records where `active: true`
2. For each user, enqueue in parallel:
   - `email-parse` task (immediate delivery)
   - `calendar-parse` task (immediate delivery)
   - `tasks-parse` task (immediate delivery)
   - `briefing-generator` task (scheduled delivery: 6:00 AM)
3. Write a `cron_run` record to Firestore (see schema below)
4. Return 200 — job complete

**Error handling:** If task enqueue fails for a user (Cloud Tasks API error), log the error, write a failed status to the `cron_run` record for that user, and continue to the next user. A failed enqueue is surfaced in the run log but does not stop fan-out for other users.

**Cron run record** — written to `cron_runs/{YYYY-MM-DD}`:

```
run_date, triggered_at, users_processed[],
users_failed[], tasks_enqueued (count), completed_at
```

---

### 3. Cloud Tasks Queues

Two queues are used — one for immediate data-source work, one for the scheduled briefing trigger.

| Queue | Delivery | Max concurrent | Used for |
|---|---|---|---|
| `data-processing` | Immediate | 10 | email-parse, calendar-parse, tasks-parse |
| `briefing-trigger` | Scheduled (6:00 AM) | 5 | briefing-generator per user |

**Retry policy for `data-processing` queue:**
- Max attempts: 3
- Backoff: 30s → 2 min → 8 min (exponential, factor 4)
- After 3 failures: task marked failed, no further retries — briefing proceeds without that data source

**Retry policy for `briefing-trigger` queue:**
- Max attempts: 3
- Backoff: 5 min → 20 min
- After 3 failures: log alert — this is the most critical failure mode

---

### 4. Data-Source Workers (Cloud Run)

Three workers, each handling one data source. All are HTTP endpoints on the same `inbox-poller` Cloud Run service (separate routes), authenticated via Cloud Tasks OIDC headers.

Each worker follows the same five-step pattern:

1. **Authenticate** — verify OIDC token from Cloud Tasks; extract `user_id` from task payload
2. **Fetch** — call the relevant external API with the user's stored OAuth token
3. **Parse** — call Gemini with item data + current Knowledge Graph context; produce structured output
4. **Write** — write parsed documents to `users/{user_id}/parsed_documents/`; write any shadow calendar items or graph updates to their respective queues
5. **Record** — update the `data_fetch_log` record for this user/date/source

**Task payload (all three workers):**

```json
{
  "user_id": "abc123",
  "date": "2026-05-07",
  "job_type": "email_parse | calendar_parse | tasks_parse"
}
```

---

#### 4a. email-parse worker

**Route:** `POST /workers/email-parse`

**Steps:**
1. Fetch unprocessed emails from shared Gmail inbox where sender matches `user.forwarding_emails[]`
2. For each email: enqueue an `email_parse` analysis job into `users/{user_id}/analysis_queue`
3. Process queue serially within this invocation — one Gemini call per email
4. Each Gemini call receives: raw email body + full Knowledge Graph (concatenated markdown)
5. Gemini returns structured JSON: `category`, `action_required`, `urgency`, `extracted_dates`, `family_member`, `shadow_calendar_items[]`
6. Write parsed document: `parsed_documents/email_{gmail_message_id}`
7. For each `shadow_calendar_item`: write to `users/{user_id}/shadow_events/`
8. Mark email as processed in Gmail (label or Firestore flag)
9. Enqueue `graph_update` jobs for any personal facts extracted

**Failure mode:** If Gemini call fails after retries, write a minimal parsed document with `raw_summary` only and `parse_failed: true`. Briefing will surface the raw subject line.

---

#### 4b. calendar-parse worker

**Route:** `POST /workers/calendar-parse`

**Steps:**
1. Fetch calendar events via Google Calendar API v3 for: today, tomorrow, and next 7 days
2. For each event: create a `cal_parse` analysis job
3. Gemini call per event: receives event data + Knowledge Graph + `[[Weekly Rhythm]]` node
4. Gemini returns: `rhythm_match` (bool), `anomaly_flag` (bool), `shadow_cal_match` (bool, checked against `shadow_events`), `conflict_flag` (bool)
5. Write parsed document: `parsed_documents/cal_{event_id}_{date}`
6. Run shadow calendar deduplication — fuzzy match against `shadow_events` for same date ± 1 hour + title similarity ≥ 0.7; write `google_cal_match` result back to the shadow event record
7. Run shadow calendar rolloff: move events where `event_date < today` to `archived_events`

---

#### 4c. tasks-parse worker

**Route:** `POST /workers/tasks-parse`

**Steps:**
1. Fetch tasks via Google Tasks API v1 — all task lists for the user
2. Filter to: overdue, due today, due within next 3 days
3. For each task: Gemini call with task data + Knowledge Graph
4. Gemini returns: `priority_assessment`, `recurrence_pattern`, `overdue` (bool), `notes_summary`
5. Write parsed document: `parsed_documents/task_{google_task_id}`
6. Also fetch previous evening capture from `evening_captures/{yesterday}` — any open items flagged as unresolved are written as synthetic task documents so the briefing generator sees them alongside real tasks

---

### 5. Briefing-Generator Task (Scheduled 6 AM)

The `briefing-trigger` queue delivers this task at 6:00 AM to the `briefing-generator` Cloud Run service. The design of the briefing generator is out of scope for this document, but from the 5 AM system's perspective:

- Task payload contains only `user_id` and `date`
- The generator queries `parsed_documents` itself — it does not receive data from the 5 AM worker directly
- The generator waits up to 5 minutes for any `analysis_queue` jobs still in `processing` state, then proceeds with what's available
- Partial data (one source failed) produces a degraded briefing with a note in the Good Morning summary: *"Calendar data wasn't available this morning — showing yesterday's schedule as a reference."*

---

## Firestore Collections Written by This System

| Collection | Written by | Purpose |
|---|---|---|
| `cron_runs/{date}` | inbox-poller | Run log — per-user status, task counts |
| `users/{id}/analysis_queue` | All workers | Per-item parse jobs |
| `users/{id}/parsed_documents` | All workers | Structured parsed output |
| `users/{id}/shadow_events` | email-parse | Dates extracted from emails |
| `users/{id}/archived_events` | calendar-parse | Rolled-off shadow events |
| `data_fetch_log/{user_id}/{date}/{source}` | All workers | Per-source fetch status, item counts, errors |

---

## Failure Modes Summary

| Failure | Behavior |
|---|---|
| inbox-poller fails to start | Cloud Scheduler retries up to 3x; if all fail, no tasks enqueued — no briefing generated. Manual re-run available. |
| Task enqueue fails for one user | Logged to `cron_run`; other users unaffected |
| Data-source task fails 3x | Marked failed in `data_fetch_log`; briefing proceeds without that source; degraded note in Good Morning section |
| Gmail OAuth token expired | email-parse fails; token refresh attempted once; if still failed, mark failed and continue |
| Briefing-generator task fails 3x | Logged as critical failure; no briefing delivered that day |

---

## Observability

- `cron_runs/{date}` — top-level daily run record; first place to check if something went wrong
- `data_fetch_log/{user_id}/{date}/{source}` — per-source detail: items fetched, items parsed, errors, duration
- Cloud Run logs — structured JSON logs per worker invocation, tagged with `user_id`, `job_type`, `date`
- Cloud Tasks console — queue depth, retry counts, failed tasks visible per queue

---

## Open Questions

**OQ-5A:** Should the inbox-poller fan-out stagger task delivery times slightly per user (e.g., user 1 at 5:00, user 2 at 5:02) to avoid thundering-herd on the Gmail and Calendar APIs, or is the user count small enough (5–10) that this isn't needed?

**OQ-5B:** OAuth token refresh — tokens are stored encrypted in Firestore per user. Should token refresh be handled inside each worker, or should there be a shared token-refresh utility in `shared/` that all workers call?

**OQ-5C:** The `briefing-trigger` queue delivers at exactly 6:00 AM. If the briefing generator takes 3–4 minutes to run, the briefing won't be ready until ~6:04. Is that acceptable, or should the scheduled delivery be pulled earlier (e.g., 5:55 AM) to target a 6:00 AM ready time?

---

*Daily Briefing System — 5 AM Cron & Queue Worker Design · April 2026 · Confidential · Jack*
