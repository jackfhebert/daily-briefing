# Daily Briefing System — Product Requirements Document
**Version 1.8 · April 2026**
**Status:** Draft — for implementation planning
**Owner:** Jack
**Last Updated:** April 2026

> **Key changes from v1.7:** User Profile, Context Summary System, and Weekly Rhythm Profile unified into a single Knowledge Graph — a set of interconnected markdown documents stored in Firestore, rooted at an `About [User]` node and branching into child nodes (people, activities, places, projects, preferences, rhythm) as topics grow. Nodes are LLM-written and split automatically when a topic is dense enough. Full graph injected into every Gemini synthesis call.

---

## Contents

1. [Overview & Vision](#1-overview--vision)
2. [Goals & Non-Goals](#2-goals--non-goals)
3. [Multi-Tenancy & User Onboarding](#3-multi-tenancy--user-onboarding)
4. [Users & Context](#4-users--context)
5. [Data Sources](#5-data-sources)
6. [Briefing Structure & Content](#6-briefing-structure--content)
7. [LLM Synthesis Engine](#7-llm-synthesis-engine)
8. [Weekly Rhythm Profile](#8-weekly-rhythm-profile)
9. [Shadow Calendar](#9-shadow-calendar)
10. [Knowledge Graph](#10-knowledge-graph)
11. [Parsed Document Store & Task Queue](#11-parsed-document-store--task-queue)
12. [Survey Submission Endpoint](#12-survey-submission-endpoint)
13. [Web Interface](#13-web-interface)
14. [System Architecture](#14-system-architecture)
15. [Phased Rollout](#15-phased-rollout)
16. [Open Questions](#16-open-questions)
17. [Success Metrics](#17-success-metrics)

---

## 1. Overview & Vision

The Daily Briefing System is a personal AI chief-of-staff that synthesizes information from a user's calendar, task list, forwarded emails, and weather into a curated, intelligent digest delivered via a web interface. The system is designed to feel less like a data aggregator and more like a thoughtful executive assistant who knows the user's priorities, family schedule, and preferences.

The system has two distinct modes: a **morning briefing** that prepares the user for the day ahead, and an **evening capture session** that helps the user offload open loops, unfinished thoughts, and pending items from memory — ensuring nothing is lost overnight and the next morning's briefing can surface what matters.

Preference learning is built in from day one through per-section micro-surveys embedded in each briefing. Survey responses are collected and stored in MVP; automated profile adaptation is a later-phase feature.

---

## 2. Goals & Non-Goals

### 2.1 Goals

- Deliver a morning briefing synthesizing calendar, todos, forwarded emails, and weather into clear prose
- Deliver an evening capture session for brain-dumping open items, thoughts, and pending tasks
- Parse forwarded emails (school, work, misc) using an LLM to extract events, action items, and context
- Generate per-section micro-survey questions in MVP; collect and store responses for later analysis
- Serve briefings as server-rendered HTML pages, one per session, retained as a permanent archive
- Run reliably on Google Cloud Run with Firestore as the data store
- Be extensible — email delivery, news, and portfolio data added in later versions

### 2.2 Non-Goals (v1)

- Email delivery of briefings (v2)
- Automated preference profile adaptation from survey data (Phase 3)
- News summaries (v2)
- Portfolio / market data (v2)
- Native mobile app

---

## 3. Multi-Tenancy & User Onboarding

The system is designed for a small number of users (5–10). It is not a public SaaS product — users are invited and provisioned manually. All data is fully isolated per user via a `user_id` scoping strategy applied consistently across Firestore and Cloud Storage.

### 3.1 Data Isolation

- Every Firestore document path is prefixed by user: `users/{user_id}/parsed_documents/...`, `users/{user_id}/knowledge_graph/...`, etc.
- Cloud Storage objects are prefixed: `briefings/{user_id}/YYYY-MM-DD/morning.html`
- All Cloud Run jobs receive a `user_id` parameter and operate strictly within that user's data scope
- The briefing generator, inbox poller, nudge sender, and queue worker all loop over active users, processing each independently
- A `users` collection in Firestore stores account metadata: `user_id`, `email`, `gmail_inbox_address`, `registered_at`, `active` (bool)

### 3.2 Email Routing

Each user forwards emails to the shared briefing Gmail inbox. Emails are routed to the correct user account by matching the sender address against the `users` collection.

- Each user record stores a `forwarding_email` — the address from which they forward emails (typically their personal Gmail)
- When the inbox poller fetches new emails, it reads the `From:` header and looks up the matching user account
- Emails from unrecognized senders are logged and ignored — no cross-user data leakage is possible
- If a user needs to forward from multiple addresses (e.g., work + personal), the user record supports a `forwarding_emails[]` array

### 3.3 User Onboarding Flow

Onboarding is intentionally lightweight. A new user is provisioned by an admin (user record created in Firestore), then directed to the onboarding page to complete setup.

1. Admin creates user record in Firestore with email and assigned `user_id`
2. User visits the system URL and authenticates via Google OAuth
3. System detects no `onboarding_complete` flag → redirects to `/onboarding`
4. User connects Google Calendar and Google Tasks (OAuth consent screen)
5. User sets their forwarding email address and confirms the briefing Gmail address to forward to
6. User writes a short self-introduction (see Section 3.4)
7. System parses the introduction with Gemini → generates root node and initial child nodes → seeds the Knowledge Graph
8. System sets `onboarding_complete: true` → redirects to dashboard
9. First briefing generated at the next scheduled 6 AM run

### 3.4 Onboarding Self-Introduction

As the final step of onboarding, the user is prompted to write a short free-text introduction — as if meeting a new executive assistant in person for the first time.

**Prompt shown to the user:**
> "Introduce yourself as if you were meeting a new assistant who will be helping organize your day. Tell them about your family, your work, your schedule, and anything else you'd want them to know to be genuinely helpful."

- No structure required — plain conversational prose is ideal
- Gemini parses the introduction and generates: (1) a root node `[[About {Name}]]` summarizing the user in prose, (2) child nodes for each named person, activity, and place mentioned. This seeds the Knowledge Graph.
- The raw introduction text is retained in the user record. It is also appended to the root node body until the graph has accumulated at least 5 nodes, at which point the graph itself provides sufficient context.
- The introduction can be edited at any time from the Preferences page

---

## 4. Users & Context

The system is designed for a small number of users (5–10). Each user has their own isolated data in Firestore and Cloud Storage, scoped by `user_id`. The initial user is a Seattle-area professional with two school-age children. Key contextual needs:

- Family calendar coordination across multiple overlapping schedules
- School and activity communications arrive via email and need to be surfaced as actionable items
- Morning briefing consumed before or during the commute — needs to be scannable and concise
- Evening capture is a low-friction habit — must feel fast and easy, not like filling out a form
- Preference for executive-style summaries, not verbose reports

---

## 5. Data Sources

### 5.1 Google Calendar

- Read via Google Calendar API v3 (OAuth 2.0)
- Fetch events for: today, tomorrow preview, and next 7 days for weekly context
- Include all calendars accessible to the authenticated user
- Surface: event title, time, location, attendees, description/notes
- Flag: scheduling conflicts, back-to-back blocks, events linked to items in the forwarded inbox

### 5.2 Google Tasks

- Read via Google Tasks API v1 — shares the same OAuth 2.0 service account as Google Calendar, no additional auth setup required
- Fetch: overdue items, due today, due within next 3 days
- Group by task list (e.g., Personal, Work, Family, Kids — user-defined lists in Google Tasks)
- Evening capture entries are stored in Firestore and surfaced alongside Google Tasks items in Section 3
- All tasks are surfaced by default regardless of due date — the preference learning system builds context over time about what matters most and handles deprioritization based on user feedback
- User can signal via section feedback ("not useful / stop surfacing this type") and the system will learn to suppress low-value recurring items over time

### 5.3 Weather

- Source: OpenWeatherMap API (free tier) or Weather.gov
- Location: Seattle area, configured via lat/long env variable
- Morning: today's forecast — high/low, precipitation probability, sunrise/sunset
- Evening capture page: tomorrow's forecast shown as context
- Smart callouts: flag rain during school pickup window (3–4 PM), weekend conditions if it's Friday

### 5.4 Forwarded Email Inbox

A general-purpose "things that matter" inbox. The user forwards any email worth knowing about — school communications, work items, invitations, reminders from others, etc.

- Dedicated Gmail account created exclusively for this system (e.g., `mybriefinginbox@gmail.com`)
- System polls inbox via Gmail API on a configurable schedule (default: every 2 hours)
- Each unprocessed email is passed to the LLM with a classification prompt to extract:
  - Category: school event / action required / FYI / invitation / deadline / other
  - Key date(s) and time(s) if present
  - Action required (yes/no) and description of the action
  - Which family member it concerns, if determinable
  - Urgency: high / normal / low
- Extracted items stored in Firestore; raw email retained for reference
- Items surface in morning briefing under "From Your Inbox" section
- All LLM calls use Gemini 1.5 Pro via Google Generative AI API
- Emails marked as processed after extraction; re-processing available manually

---

## 6. Briefing Structure & Content

The system produces two types of daily pages: a morning briefing and an evening capture. Both are server-rendered HTML, stored permanently, and accessible via the dashboard.

### 6.1 Morning Briefing

Generated fresh each morning (default: 6:00–7:00 AM). This is the primary daily artifact — an AI-synthesized digest of everything the user needs to know before the day begins.

**Section 1 — Good Morning Summary**
A 2–3 sentence LLM-written opener contextualizing the day. Surfaces the most important thing, any notable conflicts or flags, and a brief weather note if relevant. Example: *"You have back-to-back meetings from 9–12 and Emma has soccer at 4 PM. There's a 70% chance of rain this afternoon — worth factoring into pickup. One item from last night's capture is still open."*

**Section 2 — Today's Calendar**
Chronological list of today's events with time, title, and location. Flags: conflicts, back-to-back blocks, events linked to forwarded inbox items. Tomorrow's schedule shown as a compact preview below.

**Section 3 — Todos & Open Items**
Grouped list of reminders: overdue, due today, and due this week. Includes any pending items from the previous evening's brain dump that haven't been resolved. Sorted by urgency. Total open item count shown.

**Section 4 — From Your Inbox**
Items extracted from the forwarded email inbox since the last morning briefing, grouped by urgency. Action-required items flagged prominently. FYI items listed below. Source email category and relevant family member shown.

**Section 5 — Weather**
Today's forecast with practical callouts relevant to the day's schedule. Tomorrow preview shown briefly if relevant to planning (early start, outdoor activity, travel).

**Micro-Survey (Per Section)**
Each section includes a collapsed inline feedback widget. Questions are LLM-generated per section type and stored with the briefing. Responses collected and stored for future analysis (see Section 7.3).

### 6.2 Evening Capture

Available at all times at `/evening` or `/briefing/today/evening`. There is no time-gating — the user can open it at any point during the day to jot something down. At 8:00 PM, Cloud Scheduler triggers a push notification nudge reminding the user to do their end-of-day capture if they haven't already visited the page that day.

**Layout**
- Two columns on desktop, single column on mobile
- Left/top: a brief context panel — tomorrow's calendar preview, weather, and any outstanding high-urgency inbox items
- Right/bottom: the capture interface — the primary focus of the page

**Capture Interface**
A large, frictionless open-text area with the prompt: *"What's on your mind? What's open, pending, or that you don't want to forget?"*

- Auto-saves as the user types — no submit button required
- Supports free-form text — sentences, fragments, lists, anything
- Optional: quick-tap category tags (Tomorrow / This week / Someday / Remind me)
- Entries stored in Firestore under that day's evening capture record
- Next morning's briefing reads the evening capture and surfaces unresolved items in Section 3

**Evening Context Panel**
- Tomorrow's calendar — top 3–5 events
- Weather tomorrow — one-line summary
- High-urgency inbox items not yet resolved
- Count of open todos (number only — not the full list)

**Evening Micro-Survey**
A single lightweight end-of-day survey with 2–3 questions, shown at the bottom of the capture page after the user has typed something (or after 60 seconds):
- How was your day overall? (1–5 scale)
- Did this morning's briefing cover what mattered? (Yes / Mostly / No)
- Anything the system consistently misses? (open text, optional)

---

## 7. LLM Synthesis Engine

The synthesis engine is the core intelligence layer. It takes structured data from all sources and produces both the morning briefing prose and the parsed items from the forwarded inbox.

### 7.1 Responsibilities

- Write the Good Morning summary and contextual callouts
- Identify the most important items across all sections
- Detect implicit connections (e.g., rain forecast + outdoor school event = flag)
- Incorporate prior evening capture entries into the morning synthesis
- Parse forwarded emails and extract structured event/action data
- Generate per-section micro-survey questions at briefing generation time

### 7.2 Model Configuration

| Task | Model |
|------|-------|
| Morning synthesis | Gemini 1.5 Pro (Google Generative AI API) |
| Inbox email parser | Gemini 1.5 Pro (Google Generative AI API) |
| Survey question gen | Gemini 1.5 Pro |
| API keys | Stored in Google Secret Manager |

### 7.3 Prompt Architecture

- **System prompt:** user context, family info, preference profile (static in v1), output format spec
- **User prompt:** structured JSON payload of all raw data for the briefing window
- **Output:** JSON with one key per section containing rendered HTML snippet + metadata
- **Survey question generation:** separate LLM call per section, returns 2–3 multiple-choice questions as JSON
- **Email parser:** separate LLM call per email, returns structured JSON extraction

---

## 8. Weekly Rhythm Profile

The weekly rhythm profile captures the user's typical week — soft contextual knowledge about which days are usually busy, which recurring obligations fall on which days, and what the family's normal cadence looks like. The briefing uses it to provide richer context and flag when a day looks anomalous.

In v1.8, the rhythm profile is stored as the `[[Weekly Rhythm]]` node in the Knowledge Graph (Section 10), rather than as a separate JSON document. The content, update rules, and usage described below are unchanged — only the storage mechanism has moved.

### 8.1 What It Captures

- Day-level character: "Tuesdays are typically heavy — back-to-back meetings most weeks"
- Recurring family logistics: "Library books are usually due on Thursdays," "Fridays are pizza night"
- School rhythm: "Kids have early release every other Wednesday," "Soccer practice is Monday and Thursday evenings"
- Work patterns: "Usually works from home on Wednesdays," "Friday afternoons are typically clear"
- Seasonal patterns: "Ski trips usually happen on long weekends," "Q1 tends to have heavy meeting load"

### 8.2 How It Is Built

- **Passive observation:** The 5 AM queue worker scans Google Calendar and Tasks over rolling 4–8 week windows, identifying recurring patterns (same event on same weekday 3+ times). Candidates are written to the `[[Weekly Rhythm]]` node via a `graph_update` job.
- **Evening capture:** The LLM extracts statements describing recurring facts ("Fridays are always pizza night," "soccer moved to Tuesdays") and writes them to the `[[Weekly Rhythm]]` node. Corrections overwrite existing entries with a timestamp.
- 8-week calendar lookback on Phase 2 launch to bootstrap the node with observed patterns

### 8.3 How It Is Used

- Injected as part of the full Knowledge Graph into every Gemini synthesis call
- Used to add commentary: "Library books are due today" or "This Thursday looks lighter than usual"
- Anomaly flagging: when the actual calendar diverges from expected rhythm, the briefing notes it
- Cross-referenced by the shadow calendar: a school email mentioning "early release" is matched against the known pattern to assess whether it's a new deviation or the expected schedule

---

## 9. Shadow Calendar

The shadow calendar is a system-managed internal event store, separate from Google Calendar. It is populated automatically by parsing forwarded emails and other structured sources — flight confirmations, school newsletters, permission slips, event invitations — and retains events that may never make it onto the user's primary calendar. Its purpose is to act as a safety net: catching important dates that slipped through, and confirming important dates that are already tracked.

### 9.1 What Goes In

- Any date-bearing item extracted from the forwarded email inbox (school early dismissal, field trips, deadlines, recitals, flight details, hotel check-ins)
- Recurring school calendar events parsed from district newsletters or calendar attachments (.ics files forwarded to the inbox are automatically imported)
- Items mentioned in evening captures with a specific date ("don't forget Emma's recital is the 14th")
- Items are tagged by source: `school_email / flight_confirmation / invitation / user_capture / ics_import / other`

### 9.2 Deduplication Against Google Calendar

When the morning briefing generator runs, it compares shadow calendar items against Google Calendar events for the same day and time window. Three outcomes are possible:

- **Match found:** The item appears in both the shadow calendar and Google Calendar. The briefing surfaces it with a confirmation badge: "Emma's recital — 3:00 PM (confirmed by both your calendar and a school email)." This provides reassurance, not redundancy.
- **Shadow only:** The item exists in the shadow calendar but not on Google Calendar. The briefing flags it prominently: "School has early dismissal Friday — not on your calendar." This is the primary safety-net function.
- **Calendar only:** Standard calendar item, no shadow calendar confirmation. Shown normally with no badge.

### 9.3 Event Lifecycle

- Events are written to Firestore with an `event_date` field
- Events roll off automatically 24 hours after their `event_date` passes — they are archived, not deleted, so they remain in historical briefings
- Recurring events (e.g., every-other-Wednesday early release) are stored as a series with individual instances; each instance rolls off independently
- The user can dismiss a shadow calendar item from the morning briefing ("I know about this") which suppresses it from future briefings without deleting it

### 9.4 Storage Schema

**Collection:** `shadow_events` in Firestore

| Field | Description |
|-------|-------------|
| `event_date` | Date of the event |
| `title` | Event title |
| `description` | Event description |
| `source_type` | Source tag |
| `source_email_id` | Reference to source email |
| `confidence` | Confidence level |
| `family_member` | Associated family member |
| `google_cal_match` | bool |
| `dismissed` | bool |
| `archived` | bool |

**Rolloff:** Daily cleanup job (runs at 5:00 AM) moves events with `event_date < yesterday` into an `archived_events` sub-collection.

**Dedup logic:** Fuzzy match on `event_date ± 1 hour` + title similarity ≥ 0.7 (LLM-scored at briefing generation time). Match result written back to the shadow event record.

### 9.5 Relationship to Weekly Rhythm Profile

The shadow calendar and weekly rhythm profile work in tandem. The rhythm profile provides the expected pattern ("early release every other Wednesday"); the shadow calendar provides the specific confirmed instances ("early release confirmed for May 7 via school newsletter"). When both agree, the briefing can speak with high confidence. When they diverge, that divergence is itself a signal worth flagging.

---

## 10. Knowledge Graph

The Knowledge Graph is the unified context model for everything the system knows about a user. It replaces the previously separate User Profile, Context Summary, and Weekly Rhythm Profile with a single structured representation: a graph of interconnected markdown documents, rooted at a central article about the user and branching into sub-nodes as topics accumulate enough detail to warrant their own page.

The design is modeled on a personal wiki. Each node is a short markdown article, written in plain prose, with wiki-style links to other nodes (`[[Emma]]`, `[[Soccer]]`, `[[Glacier Trip 2026]]`). The root node is always `[[About Jack]]` — a well-written overview of who the user is. Everything else hangs off it.

### 10.1 Graph Structure

| Field | Details |
|-------|---------|
| **Storage** | Firestore collection: `users/{user_id}/knowledge_graph/`. One document per node. |
| **Document fields** | `slug` (e.g., `about_jack`, `emma`, `soccer`), `title`, `body` (markdown text), `links[]` (outbound slugs), `node_type` (`root \| person \| activity \| place \| project \| preference \| rhythm`), `created_at`, `updated_at`, `source_refs[]` |
| **Root node** | slug: `about_{user_first_name}` — created from the onboarding self-introduction. The entry point for all graph traversal. |
| **Link syntax** | Standard wiki-link format: `[[Node Title]]` in the markdown body. Resolved to slugs at read time. |

### 10.2 Node Types & Examples

- `root` — `[[About Jack]]`: employer, location, family overview, general schedule, stated priorities. Links to all major sub-nodes.
- `person` — `[[Emma]]`: age, grade, school, activities, personality notes, relevant recurring events. `[[Liam]]`: similar. `[[Sarah (spouse)]]`: role, schedule patterns.
- `activity` — `[[Soccer]]`: which child, practice days, coach name, season dates, carpool arrangements. `[[Piano Lessons]]`: day/time, teacher, upcoming recitals.
- `place` — `[[Lincoln Elementary]]`: address, office number, known early dismissal pattern, key contacts.
- `project` — `[[Glacier Trip 2026]]`: dates, participants, open tasks, confirmed bookings, outstanding items.
- `preference` — `[[Briefing Preferences]]`: tone, verbosity per section, what to emphasize, what to skip. Fed by survey insights.
- `rhythm` — `[[Weekly Rhythm]]`: day-of-week patterns, recurring soft facts (library books Thursdays, pizza Fridays), observed calendar patterns.

### 10.3 How the Graph Is Built

The graph grows from four sources, all writing to it asynchronously via the analysis queue:

- **Onboarding introduction:** Gemini parses the self-introduction and generates the root node and any immediately obvious child nodes (one per named person, one per named activity). This is the cold-start seed.
- **Forwarded email parsing:** the inbox parser extracts personal facts — a child's name, a school, a coach — and writes them as updates to the relevant node, or as candidates for a new node if the entity doesn't exist yet.
- **Evening brain dump:** the capture parser scans for statements that reveal new facts or update known ones ("Emma got moved up to the advanced soccer group") and updates the appropriate node.
- **Survey and profile question answers:** each answered question generates a 2-sentence insight that is written to the `[[Briefing Preferences]]` node or the most semantically relevant node.

### 10.4 Automatic Node Splitting

When a topic accumulates enough detail within a parent node's body, Gemini automatically splits it into a new child node. This happens as part of the `graph_update` job in the analysis queue.

- After any graph write, a lightweight Gemini call evaluates whether any inline section of the updated node has grown beyond ~300 words or contains 5+ distinct facts
- If so, Gemini drafts a new node document from that content, updates the parent to replace the inline section with a `[[New Node]]` link, and writes both documents to Firestore
- The split is logged in `graph_update_log` so it can be reviewed from the Preferences page
- No user confirmation required — the split happens automatically. The user can review and merge nodes manually from the Preferences page if desired.

### 10.5 LLM Injection

The full graph is injected into every Gemini synthesis call as a single concatenated markdown block. Nodes are ordered: root first, then child nodes in breadth-first order, separated by `---` dividers. Each node's wiki-links are preserved so the LLM can understand the relational structure.

- Injected under the heading: *"Here is everything we know about the user, structured as a personal knowledge graph:"*
- The preference node (`[[Briefing Preferences]]`) is always injected last, immediately before the data payload, so its instructions are closest to the synthesis task
- Future optimization (Phase 3+): selective injection — load root + nodes referenced by today's briefing data only

### 10.6 Relationship to Rhythm Profile & Shadow Calendar

The Weekly Rhythm Profile is now the `[[Weekly Rhythm]]` node within the graph. The Shadow Calendar remains a separate Firestore collection (it is event-specific and time-indexed, not narrative knowledge), but events extracted from emails may write contextual facts back into relevant graph nodes.

---

## 11. Parsed Document Store & Task Queue

Rather than parsing raw data inline at briefing generation time, the system pre-processes every input item into a named, structured document during the 5 AM cron window. The 6 AM briefing generator then assembles pre-computed outputs — making synthesis faster, more reliable, and independently debuggable.

### 11.1 Parsed Document Store

Every item that passes through the system is parsed by an LLM and stored as a structured document in Firestore.

**Collection:** `parsed_documents` in Firestore

| Field | Description |
|-------|-------------|
| **Document ID scheme** | Typed + source-based: `email_{gmail_message_id}`, `task_{google_task_id}`, `cal_{event_id}_{date}`, `capture_{YYYY-MM-DD}` |
| **Common fields** | `doc_type`, `source_id`, `raw_summary`, `structured_data` (JSON), `relevance_score`, `flags[]`, `family_member`, `date_refs[]`, `parsed_at`, `context_used` |
| **Email document** | + `sender`, `subject`, `category`, `action_required`, `urgency`, `extracted_dates`, `shadow_calendar_items[]` |
| **Task document** | + `task_list`, `due_date`, `overdue` (bool), `recurrence_pattern`, `priority_assessment`, `notes_summary` |
| **Calendar document** | + `event_title`, `time`, `location`, `attendees`, `shadow_cal_match` (bool), `rhythm_match` (bool), `anomaly_flag` (bool) |
| **Retention** | Documents older than 30 days are archived to a `parsed_documents_archive` sub-collection |

### 11.2 Task Queue

The 5 AM cron enqueues one analysis job per item into a Firestore-backed task queue. Each job is processed independently — failures are isolated, jobs can be retried, and the overall pipeline is observable.

**Collection:** `analysis_queue` in Firestore

| Field | Description |
|-------|-------------|
| `job_id` | Auto-generated |
| `job_type` | `email_parse \| task_parse \| cal_parse \| capture_parse \| graph_update \| graph_split_check` |
| `source_ref` | Reference to source item |
| `status` | `queued \| processing \| complete \| failed` |
| `created_at` | Timestamp |
| `completed_at` | Timestamp |
| `output_doc_id` | Reference to output document |
| `error_message` | Set on failure |

**Worker:** The `inbox-poller` Cloud Run job acts as both the enqueuer (5 AM) and the worker — it processes its own queue serially after enqueuing, with a per-job timeout.

**Retry policy:** Failed jobs are retried once automatically. After two failures, the job is marked `failed` and a fallback raw summary is written so the briefing can still include the item with reduced richness.

### 11.3 Briefing Generator Assembly

At 6 AM, the briefing generator does not call parsing LLMs directly. Instead it:

1. Queries `parsed_documents` for all documents with `date_refs` relevant to today and the next 7 days
2. Loads the full Knowledge Graph (all nodes from `users/{user_id}/knowledge_graph/`), concatenated in breadth-first order
3. Checks `analysis_queue` for any jobs still in `processing` or `queued` state — waits up to 5 minutes, then proceeds with available documents
4. Assembles a single rich JSON payload of all parsed documents + context models
5. Makes one synthesis LLM call to produce the full briefing — no parsing happens at this stage
6. Makes per-section survey question generation calls

### 11.4 Document Naming Examples

| Document ID | Description |
|-------------|-------------|
| `email_18f3a2b9c1d` | School newsletter — category=school_event, early dismissal date May 7, shadow_calendar_item written, action_required=false |
| `task_abc123_family` | Google Task from Family list — priority_assessment=high (overdue 3 days), recurrence_pattern=weekly_thursday |
| `cal_xyz789_2026-05-07` | Calendar event May 7 — rhythm_match=true, shadow_cal_match=true, anomaly_flag=false |
| `capture_2026-05-06` | Evening capture May 6 — extracted rhythm entries (2), shadow events (1: "Emma's recital May 14"), open_items (3) |

---

## 12. Survey Submission Endpoint

When a user answers a survey question, the response is submitted via a POST request to a dedicated Cloud Run endpoint. The endpoint logs and persists immediately, then hands off to the async analysis queue.

### 12.1 Endpoint Spec

| Field | Details |
|-------|---------|
| **Route** | `POST /api/survey/submit` |
| **Auth** | Session cookie (Google OAuth) — request rejected with 401 if unauthenticated. `user_id` extracted from session, never from request body. |
| **Request body** | `{ briefing_id, section, question_id, question_text, answer, answer_index, submitted_at }` |
| **Response** | `200 { status: "logged" }` — returned immediately after steps 1–2, before any LLM work begins |
| **Hosted on** | `web-server` Cloud Run service |

### 12.2 Processing Steps

1. Authenticate: validate session cookie; extract and verify `user_id`
2. Log & persist: write response document to `users/{user_id}/survey_responses/{response_id}` in Firestore. Return `200 { status: "logged" }` to client immediately.
3. Enqueue analysis job: write a job record to `users/{user_id}/analysis_queue` with `job_type: "graph_update"`, `source_ref: response_id`, `status: "queued"`
4. Worker picks up job (asynchronously): reads the survey response, calls Gemini to generate a 2-sentence insight, writes it to the most semantically relevant knowledge graph node. Triggers a `graph_split_check` job on the updated node.

Steps 1–2 are synchronous (milliseconds). Steps 3–4 are fully async — the user never waits for them.

### 12.3 Evening Profile Question Submission

Targeted profile questions on the evening capture page use the same endpoint with `section: "user_profile"`. The analysis job type is still `graph_update` — the worker writes the extracted fact to the most relevant graph node and triggers a `graph_split_check`.

### 12.4 Client-Side Behavior

- Survey widget submits on selection — no confirm button needed for single-choice questions
- On successful 200: widget collapses and displays a checkmark immediately
- On network error or non-200: widget shows a quiet retry option; does not block the page
- Submissions are idempotent — duplicate POSTs for the same `question_id` within the same `briefing_id` are deduplicated at the server

---

## 13. Web Interface

### 13.1 Rendering Approach

All pages are server-rendered HTML. The Cloud Run web service generates a complete HTML file for each briefing session at generation time and writes it to Cloud Storage. Pages are served directly — no client-side rendering framework required.

- Each morning briefing is a self-contained HTML file, stored and served as a permanent page
- URL structure: `/briefing/YYYY-MM-DD/morning` and `/briefing/YYYY-MM-DD/evening`
- `/` (root) redirects to the most recent briefing page
- Minimal JavaScript: only for the capture textarea autosave and survey widget interactions

### 13.2 Page Structure

| Page | Layout |
|------|--------|
| **Morning briefing** | Header with date/greeting → sections (Summary, Calendar, Todos, Inbox, Weather) → micro-surveys per section → footer with archive link |
| **Evening capture** | Two-panel layout: context panel (calendar/weather/inbox) + capture textarea and 2–3 targeted profile questions → end-of-day survey → footer |
| **Archive index** | `/archive` — reverse-chronological list of all past briefing days with links to morning and evening pages |
| **Preferences** | `/preferences` — view current preference profile, user profile facts, and context summary entries; manual edits supported in v1 |

### 13.3 Per-Section Micro-Surveys

- Survey widget appears collapsed beneath each section by default
- Expands with a single tap/click — no page navigation
- Questions are multiple-choice (2–4 options) — no typing required
- 2–3 questions per section, LLM-generated based on section type and content
- On submit: response stored in Firestore + context summary insight generated and appended
- Visual confirmation on submit — widget collapses and shows a checkmark

### 13.4 Evening Profile Questions

- 2–3 multiple-choice questions generated fresh each evening based on gaps in the Knowledge Graph
- Questions are shown above or beside the capture textarea, not in a separate flow
- Answering is optional — questions disappear once answered and never repeat
- Each answer triggers a context summary insight generation call (same mechanism as morning surveys)
- The open-text brain dump area is always present — profile facts extracted from it passively

### 13.5 Design Principles

- Mobile-first — primary consumption is on a phone
- Readable at a glance — generous type size, clear hierarchy, no clutter
- Dark mode support via CSS `prefers-color-scheme`
- No external JS frameworks or CDN dependencies — fully self-contained HTML+CSS+vanilla JS
- Fast load — pages are pre-rendered; no API calls on page load except evening capture autosave
- Authentication: Google OAuth; each user tied to their Google account; session cookie with 30-day expiry

---

## 14. System Architecture

### 14.1 Components

| Component | Description |
|-----------|-------------|
| **briefing-generator** | Cloud Run job — triggered at 6 AM. Loops over active users; reads pre-computed `parsed_documents` and context models, makes one Gemini synthesis call, generates survey questions, renders HTML, writes to `briefings/{user_id}/` in Cloud Storage and Firestore metadata. |
| **inbox-poller / queue-worker** | Cloud Run job — triggered at 5 AM. Fetches new emails from shared briefing inbox; routes each to the correct user by sender address; enqueues analysis jobs per user; fetches tasks and calendar events per user via per-user OAuth tokens; processes `analysis_queue`; writes `parsed_documents`. |
| **web-server** | Cloud Run service — serves morning HTML pages, evening capture page, archive index, preferences, onboarding flow, and `POST /api/survey/submit` endpoint. All routes authenticated; data scoped by `user_id` from session. |
| **nudge-sender** | Cloud Run job — triggered at 8 PM. Loops over active users; checks each for evening page visit; sends Gmail nudge if not visited. |
| **Firestore** | All collections nested under `users/{user_id}/`: `knowledge_graph`, `parsed_documents`, `parsed_documents_archive`, `analysis_queue`, `shadow_events`, `archived_events`, `inbox_items`, `evening_captures`, `survey_responses`, `briefing_metadata`, `graph_update_log`. Top-level `users` collection stores account records. |
| **Cloud Storage** | HTML briefing archive at `briefings/{user_id}/YYYY-MM-DD/{morning\|evening}.html` |
| **Secret Manager** | Google AI (Gemini) API key, OpenWeatherMap API key. Per-user OAuth tokens stored encrypted in Firestore. |

### 14.2 Integrations

| Integration | Details |
|-------------|---------|
| Google Calendar | Google Calendar API v3 — per-user OAuth 2.0; tokens stored per user in Firestore |
| Google Tasks | Google Tasks API v1 — same per-user OAuth token as Calendar |
| Gmail (read) | Gmail API — read-only access to shared briefing inbox; emails routed to users by sender address match |
| Gmail (send) | Gmail API — send from shared briefing inbox for nudges and Phase 4 email delivery |
| Weather | OpenWeatherMap API — lat/long stored per user in Firestore user record |
| LLM — Gemini | Gemini 1.5 Pro via Google Generative AI API — all synthesis, parsing, survey generation, context summary, and analysis queue jobs |

### 14.3 Morning Briefing Data Flow

**Triggered 6:00 AM daily:**

1. Cloud Scheduler triggers `briefing-generator` at 6:00 AM
2. Check `analysis_queue` — wait up to 5 min for any jobs still in processing to complete
3. Query `parsed_documents` for all documents relevant to today + next 7 days
4. Load the full Knowledge Graph (all nodes from `users/{user_id}/knowledge_graph/`), concatenated breadth-first
5. Assemble data payload: all parsed documents + full knowledge graph markdown + shadow calendar flags
6. One Gemini synthesis call: system prompt (full knowledge graph) + data payload → JSON with HTML per section
7. Per-section survey question generation calls
8. Render final HTML; write to Cloud Storage and Firestore `briefing_metadata`

### 14.4 5 AM Cron / Queue Worker Data Flow

**Triggered 5:00 AM daily:**

1. Cloud Scheduler triggers `inbox-poller / queue-worker` at 5:00 AM
2. Fetch new emails (Gmail API), updated tasks (Google Tasks API), today's calendar events (Google Calendar API)
3. For each item: create a job record in `analysis_queue` with `status: queued`
4. Process queue serially: for each job, call parser LLM with item data + current context models
5. Write parsed document to `parsed_documents`; update job record to `complete` with `output_doc_id`
6. During parsing: extract shadow calendar items → `shadow_events`; extract personal facts and rhythm patterns → `graph_update` jobs targeting relevant knowledge graph nodes
7. Run shadow calendar rolloff: move past events to `archived_events`
8. Run rhythm bootstrap if `[[Weekly Rhythm]]` node has < 5 entries (8-week calendar lookback)
9. All `parsed_documents` ready for `briefing-generator` at 6:00 AM

### 14.5 Knowledge Graph Update Flow

**Triggered on every answered survey or evening profile question:**

1. User submits an answer to a multiple-choice question (morning survey or evening profile question)
2. Response stored in Firestore `survey_responses`
3. LLM call: "Given this question and answer, write a 2-sentence insight for a personal briefing system"
4. Generated insight written to most relevant knowledge graph node via `graph_update` job
5. `graph_split_check` job triggered on updated node — Gemini evaluates whether node should be split

### 14.6 Evening Capture Data Flow

**User-initiated, available at all times:**

1. User navigates to `/evening`
2. Evening-server fetches context panel data + loads Knowledge Graph to generate 2–3 gap-filling questions
3. Renders page: context panel + targeted profile questions + capture textarea
4. User types — autosave fires on keyup debounce (500ms); raw text written to Firestore
5. On save, evening-server enqueues a `capture_parse` job in `analysis_queue`
6. Queue worker processes job: extracts shadow calendar events → `shadow_events`; extracts personal/contextual facts → writes to relevant knowledge graph nodes via `graph_update` jobs
7. User answers a profile question → survey endpoint → `graph_update` job enqueued

### 14.7 Evening Nudge Flow

**Triggered daily at 8:00 PM:**

1. Cloud Scheduler triggers `nudge-sender` at 8:00 PM
2. Check Firestore: has the user visited the evening capture page today?
3. If not visited: send nudge email via Gmail API (send)
4. If already visited: skip

---

## 15. Phased Rollout

### Phase 1 — Foundation (MVP)

- Cloud Run infra, CI/CD, Secret Manager
- Firestore schema: all collections under `users/{user_id}/`; top-level `users` collection
- Google OAuth + admin user provisioning flow
- Onboarding: Calendar/Tasks OAuth, forwarding email setup, self-introduction → Gemini parse → Knowledge Graph seed (root node + initial child nodes)
- Google Calendar + Weather (per-user OAuth)
- Task queue: `analysis_queue` + `parsed_documents`
- 5 AM cron: enqueue + process calendar/task parse jobs per user
- Morning briefing generator: reads `parsed_documents` + full knowledge graph, Gemini synthesis, server-rendered HTML to `briefings/{user_id}/`
- Web server + archive + survey submission endpoint (`POST /api/survey/submit`)
- Survey → log + persist → enqueue `graph_update` job (async) → on completion, enqueue `graph_split_check` job

### Phase 2 — Full V1

- Google Tasks (all lists) — parse jobs enqueued at 5 AM
- Gmail inbox polling → sender-address routing → parse jobs → shadow calendar + user profile
- Shadow calendar deduplication + confirmation badges
- Weekly rhythm profile — 8-week bootstrap + passive observation
- Evening capture: autosave + `capture_parse` job → profile writes
- Targeted knowledge graph gap-filling questions on evening page → survey endpoint → `graph_update` job
- Evening nudge via Gmail send (per-user, skip if already visited)
- Full knowledge graph injected into Gemini synthesis; graph grows continuously from all sources
- Preferences page — view profiles, context summary, parsed document archive

### Phase 3 — Learning Loop

- Selective graph injection — load root + day-relevant nodes only (replaces full-graph injection for efficiency)
- Profile change notifications in briefing
- Context summary consolidation pass
- Rhythm profile decay + stale entry management
- Survey response trends dashboard

### Phase 4 — V2 Features

- Morning briefing email delivery via Gmail API
- News summary section
- Portfolio / market pulse section
- Mark todos complete from dashboard
- Multi-day trip awareness & prep checklists
- Agentic capability: system can research and act on user's behalf — check flight prices, find shopping links, summarize articles, look up product options. Tasks requested via evening capture or morning briefing action items; results delivered in next morning's briefing.

---

## 16. Open Questions

| ID | Question |
|----|----------|
| **OQ-1** | Cloud Storage vs. Firestore for HTML page storage — Recommended split: HTML in Cloud Storage (CDN-backed), metadata in Firestore. Confirm at implementation. |
| **OQ-2** | Evening capture archive — confirm whether old evening pages should be browseable in the archive alongside morning briefings, or stored only as raw Firestore data. |
| **OQ-3** | Google Tasks list structure — confirm which task lists exist and should be synced. System surfaces all by default; user tunes via feedback. |
| **OQ-4** | Queue worker parallelism — the 5 AM worker processes jobs serially by default. If the number of daily emails + tasks grows large, evaluate moving to parallel Cloud Run jobs or Cloud Tasks for concurrency. Monitor job completion time in Phase 2. |
| **OQ-5** | Agentic task scope (Phase 4) — define which categories of agentic tasks are in scope for v2: flight search, product links, article summaries, calendar booking. Each has different tool requirements and latency profiles. Scope this before Phase 4 planning. |
| **OQ-6** | Gemini model version — Gemini 1.5 Pro is the default. Evaluate Gemini 2.0 Flash for lower-latency queue worker jobs (parsing, context summary generation) where speed matters more than reasoning depth. |

---

## 17. Success Metrics

- Morning briefing opened 5+ days/week within 30 days of launch
- Evening capture used 3+ times/week within 30 days of launch
- Zero missed school or family events attributable to system failure
- Briefing generation success rate > 99% for scheduled runs
- Shadow calendar catches ≥ 1 missed event in first 30 days
- Weekly rhythm profile accumulates ≥ 5 entries within 30 days of Phase 2
- Knowledge Graph contains ≥ 5 nodes within 14 days of Phase 2 launch
- Knowledge Graph root node updated ≥ 10 times within 60 days, with at least 3 auto-split child nodes created
- Survey responses collected for > 70% of morning briefings within 60 days
- Phase 3: measurable preference profile change in ≥ 2 sections within 30 days of analysis pipeline launch

---

*Daily Briefing System PRD v1.8 · April 2026 · Confidential · Jack*
