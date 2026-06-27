# Daily Briefing System
## Email-Based User Onboarding — System Design

**Status:** Design complete — ready for implementation planning
**Last Updated:** April 2026
**Author:** Jack

> **Relationship to the PRD:** this design covers a separate, email-first intro flow — a user emails `jhebert-bot@gmail.com` to start building their Knowledge Graph asynchronously, independent of the web-based onboarding in [PRD §3.3](../PRD.md#33-user-onboarding-flow). The handoff between this flow and `onboarding_complete` is now designed in [Web-Based Onboarding & Signup — System Design §4](./web-onboarding.md#4-reconciling-with-the-email-first-intro-flow).

---

## Overview

Users send an email to `jhebert-bot@gmail.com` with subject **"Intro"** to begin onboarding. They can reply to that thread at any time to add or correct information. A cron job detects new activity, queues processing tasks, and Gemini incrementally builds and updates the user's Knowledge Graph and shadow schedule over time.

---

## Components

### 1. Cron Job — `intro-poller`

**Trigger:** Cloud Scheduler, every 2 hours

**Responsibility:** Detect unprocessed user messages in Intro threads and enqueue tasks.

**Logic:**

1. Call Gmail API — search for all threads matching subject `"Intro"` in `jhebert-bot@gmail.com`
2. For each thread, fetch all messages
3. Filter to messages where sender is *not* `jhebert-bot@gmail.com` (i.e., user messages only)
4. For each user message, check Firestore `processed_messages` collection — skip if message ID already present
5. For any unprocessed user messages found, enqueue one task per **thread** into `intro_processing_queue` — the task payload includes the thread ID and the list of new unprocessed message IDs found in this run
6. If multiple unprocessed messages exist in the same thread, bundle them into one task (processed together, in chronological order)

**Why one task per thread, not per message:** Gemini needs the full existing profile context plus all new messages together to do a coherent incremental update. Splitting messages into separate tasks risks race conditions on the profile.

---

### 2. Task Queue — `intro_processing_queue` (Firestore)

| Field | Type | Description |
|---|---|---|
| `task_id` | string | Auto-generated |
| `thread_id` | string | Gmail thread ID |
| `user_email` | string | Sender address — used to look up or create the user |
| `new_message_ids` | string[] | The unprocessed message IDs to act on this run |
| `status` | enum | `queued / processing / complete / failed` |
| `created_at` | timestamp | |
| `completed_at` | timestamp | |
| `error_message` | string | Set on failure |
| `gemini_output_ref` | string | Reference to the stored Gemini JSON output |

---

### 3. Task Worker — `intro-worker`

Processes one task at a time from `intro_processing_queue`. This is the core of the design.

#### Step 1 — Resolve the User

Look up the sender email in the `users` Firestore collection.

- **Found:** load their existing Knowledge Graph nodes and shadow schedule entries
- **Not found:** create a new user record (stub — no `onboarding_complete` flag yet), initialize empty Knowledge Graph and shadow schedule

#### Step 2 — Fetch Message Content

For each message ID in `new_message_ids`:
- Fetch the full message body from Gmail
- Strip any quoted reply text (only process what the user wrote in *this* message, not re-quoted prior content)
- Assemble messages in chronological order

#### Step 3 — Build the Gemini Prompt

Two inputs are assembled:

**Existing context** (may be empty for a new user):
- The current `about_{user}` root node markdown (if it exists)
- The current shadow schedule entries (if any)
- A summary of clarifying questions previously asked (so Gemini doesn't repeat them)

**New content:**
- The new user message(s), in chronological order

**Persona instruction:** Gemini is instructed to behave as a thoughtful executive assistant onboarding a new principal — warm, professional, perceptive. It should read between the lines, infer unstated structure, and treat the user's words as a conversation, not a form to fill out.

#### Step 4 — Gemini Call

Gemini is called once and returns a single structured JSON object (see schema below). All analysis, profile writing, and question generation happens in this call.

#### Step 5 — Apply Profile Updates

For each node in `profile_updates`:
- `create` → write new Firestore document under `users/{user_id}/knowledge_graph/`
- `update` → write full updated body (Gemini has been given existing content, so its output is the complete updated node — no partial patching)
- `none` → skip

Apply the same create/update/none logic to shadow schedule entries.

#### Step 6 — Mark Messages as Processed

For each message ID in `new_message_ids`, write a record to `processed_messages` (see schema below). This is the deduplication record checked by the cron job.

#### Step 7 — Enqueue Reply (if warranted)

If `reply_warranted: true` and `clarifying_questions` is non-empty, write a job to `outbound_email_queue` with the questions and thread ID.

> **Note:** Actual email sending is not implemented yet. The `outbound_email_queue` collection is the stub for the future send feature. The worker does not send email directly.

---

## Gemini Output Schema

Gemini returns a single JSON object. The worker applies it — Gemini does not write to Firestore directly.

```json
{
  "profile_updates": {
    "root_node": {
      "action": "create | update | none",
      "content": "<full markdown body for the about_{user} node>"
    },
    "child_nodes": [
      {
        "slug": "emma",
        "title": "Emma",
        "node_type": "person | activity | place | project | preference | rhythm",
        "action": "create | update | none",
        "content": "<full markdown body>"
      }
    ]
  },
  "shadow_schedule_entries": [
    {
      "action": "create | update | none",
      "title": "Soccer practice",
      "recurrence": "Monday and Thursday evenings",
      "family_member": "Emma",
      "confidence": "high | medium | low",
      "source": "user_stated | inferred"
    }
  ],
  "clarifying_questions": [
    "What school do your kids attend?",
    "Do you work from home on any regular days?"
  ],
  "reply_warranted": true,
  "reasoning": "<brief internal note on what changed and why — not sent to user>"
}
```

**Schema notes:**

- `action` on each node makes Gemini's intent explicit — the worker applies it, preventing accidental clobbers
- `clarifying_questions` is capped at 3; empty array and `reply_warranted: false` if nothing to ask
- `confidence` on shadow schedule entries flags inferred vs. stated facts, useful later for the rhythm profile
- `reasoning` is a scratchpad for debuggability — never shown to the user

---

## Firestore Collections

### `processed_messages`

Keyed by Gmail message ID. The primary deduplication mechanism.

| Field | Type | Description |
|---|---|---|
| `message_id` | string | Gmail message ID (document key) |
| `thread_id` | string | Gmail thread ID |
| `user_email` | string | Sender address |
| `processed_at` | timestamp | When the worker completed processing |
| `task_id` | string | Back-reference to the queue task |

### `outbound_email_queue`

Stub collection for future reply sending.

| Field | Type | Description |
|---|---|---|
| `job_id` | string | Auto-generated |
| `thread_id` | string | Gmail thread to reply to |
| `user_email` | string | User to reply to |
| `questions` | string[] | Clarifying questions to include |
| `status` | enum | `queued / sent / failed` |
| `created_at` | timestamp | |

### Collections Summary

| Collection | Purpose |
|---|---|
| `users` | User account records, looked up by email |
| `users/{id}/knowledge_graph/` | Markdown nodes — root + children |
| `users/{id}/shadow_schedule/` | Recurring schedule entries seeded from intro |
| `intro_processing_queue` | Task queue for the worker |
| `processed_messages` | Deduplication log, keyed by Gmail message ID |
| `outbound_email_queue` | Stub for future reply sending |

---

## Test Endpoint — `POST /api/dev/intro-simulate`

Available in **dev/staging environments only** — blocked in production by env flag.

**Purpose:** Feed raw thread data directly into the worker logic and get the Gemini JSON output back, without needing real Gmail threads or a real user record.

### Request Body

```json
{
  "user_email": "test@example.com",
  "messages": [
    {
      "from": "test@example.com",
      "body": "Hi, I'm Jack. I have two kids, Emma (8) and Liam (6)...",
      "sent_at": "2026-04-25T09:00:00Z"
    }
  ],
  "existing_profile": {
    "root_node": null,
    "child_nodes": []
  },
  "existing_shadow_schedule": []
}
```

### Response

The raw Gemini JSON output, plus `dry_run: true` confirming nothing was written to Firestore.

### Test Scenarios to Cover

- First introduction from a new user (no existing profile)
- Follow-up reply adding new information
- Reply that corrects or contradicts existing profile data
- Ambiguous input that warrants clarifying questions
- Thread with multiple unprocessed user messages bundled together

---

## Thread Processing — Flow Diagram

```
Gmail thread: subject "Intro"
│
├── msg_001  from: jack@gmail.com        ← user message, unprocessed → enqueue
├── msg_002  from: jhebert-bot@gmail.com ← ours, always skip
├── msg_003  from: jack@gmail.com        ← user message, unprocessed → bundle with msg_001
│
Worker receives task: thread_id, new_message_ids: [msg_001, msg_003]
│
├── 1. Resolve user (look up or create)
├── 2. Fetch msg_001 and msg_003 bodies, strip quoted text
├── 3. Load existing profile + prior questions asked
├── 4. Call Gemini → structured JSON
├── 5. Apply profile_updates to knowledge_graph/
├── 6. Apply shadow_schedule_entries to shadow_schedule/
├── 7. Write msg_001, msg_003 to processed_messages
└── 8. If reply_warranted → write to outbound_email_queue
```

---

## Deliberately Deferred

The following are out of scope for this design phase and will be addressed separately:

- **Actually sending reply emails** — the `outbound_email_queue` stub is the seam for this; a future worker reads from it and calls the Gmail send API
- ~~**Promoting a user to full onboarding**~~ — now designed; see [`web-onboarding.md` §1 and §4](./web-onboarding.md)
- **Knowledge Graph node splitting** — the `graph_split_check` logic from PRD §10.4 applies eventually, but the intro worker does not trigger it
- **Rate limiting / abuse prevention** — emails from unrecognized senders are ignored per existing PRD routing logic; additional hardening deferred

---

*Daily Briefing System — Email Onboarding Design · April 2026 · Confidential · Jack*
