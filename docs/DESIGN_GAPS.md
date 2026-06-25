# Remaining Design Work — Path to MVP

**Purpose:** Track which system design docs still need to be written before implementation can start on the Phase 1 MVP (per [PRD §15](./PRD.md#phase-1--foundation-mvp)). This doc is a living checklist, not a design itself — each open item below should eventually become its own doc, most likely under [`docs/designs/`](./designs/).

---

## 1. What's already designed

| Doc | Covers | Leaves open |
|---|---|---|
| [`designs/email-onboarding.md`](./designs/email-onboarding.md) | Email-first intro flow (`jhebert-bot@gmail.com`, "Intro" thread) that incrementally builds the Knowledge Graph and shadow schedule from email replies | Handoff to `onboarding_complete` / the web-based onboarding flow in PRD §3.3 is explicitly deferred |
| [`designs/cron-queue-worker.md`](./designs/cron-queue-worker.md) | 5 AM Cloud Tasks fan-out: email-parse, calendar-parse, tasks-parse, and weather-fetch workers, plus the guaranteed 6 AM briefing-generator trigger | Briefing generator itself is explicitly out of scope; OAuth token refresh is an open question (OQ-5B) |

Both existing design docs *reference* `graph_update` and `graph_split_check` jobs (written by the email-intro worker, the 5 AM workers, and the survey endpoint per PRD §12.2) — but no doc yet describes the worker that actually consumes those jobs. That gap is called out as its own item below (§3.3).

---

## 2. Scope conflicts — resolved (PRD v1.9)

Both mismatches flagged in earlier revisions of this doc are now resolved:

- **Email parsing is now Phase 1.** PRD §15 was updated to pull Gmail inbox polling into Phase 1, matching what `cron-queue-worker.md` already designed. As a consequence, shadow-calendar extraction + deduplication (which depends on email-parse writing `shadow_events` for calendar-parse to dedup against) also moved to Phase 1 — Phase 2 now only covers the remaining UI polish (confirmation badges).
- **Weather now has a worker.** `cron-queue-worker.md` §4d adds a `weather-fetch` worker to the 5 AM fan-out. It fetches a per-user `tracked_cities[]` list (mirroring `forwarding_emails[]`) against the NWS API — no API key, no per-item Gemini parse, just a cached gridpoint lookup + forecast call per city. PRD §5.3 was rewritten to match.

---

## 3. Design docs still needed for MVP

These map directly to PRD §15 Phase 1 bullets that don't yet have a system design. Suggested order reflects dependencies (e.g., the briefing generator can't be fully designed until the Knowledge Graph read/write contract is settled).

### 3.1 Briefing Generator (highest priority)
The core 6 AM service and the actual deliverable of the MVP — currently explicitly out of scope in `cron-queue-worker.md`. Needs: how it assembles the data payload from `parsed_documents` + the full Knowledge Graph, the synthesis prompt structure (system prompt vs. data payload, per PRD §7.3), how it waits on in-flight `analysis_queue` jobs, how it produces the per-section HTML + survey questions in one or more Gemini calls, and how/where it writes the final HTML (Cloud Storage path, Firestore `briefing_metadata` record).

### 3.2 Web-Based OAuth Onboarding Flow
PRD §3.3 describes the steps (OAuth consent, forwarding-email setup, self-introduction form) but there's no system design for the actual onboarding web app — pages, OAuth callback handling, where the self-introduction text is captured and handed to Gemini for the initial Knowledge Graph seed. This is distinct from `email-onboarding.md`'s email-first flow; the two need a defined relationship (does a user do one, the other, or both? does completing one short-circuit the other?).

### 3.3 Knowledge Graph Engine (`graph_update` / `graph_split_check` workers)
Three different sources (email-intro worker, 5 AM parse workers, survey endpoint) all enqueue `graph_update` jobs, but no doc describes the worker that processes them: how it picks the "most semantically relevant node" for a given fact, how it writes the update (full-body replace, per PRD §10.3), and the splitting logic in `graph_split_check` (the ~300-word / 5-fact threshold from PRD §10.4, how the split is drafted and logged). This is shared infrastructure underpinning almost everything else, so worth designing right after the briefing generator.

### 3.4 Web Server (page rendering, auth, archive, survey endpoint)
PRD §12 specs the survey endpoint contract and §13 specs page structure, but there's no design for the `web-server` Cloud Run service itself: routing, session/auth (Google OAuth login + 30-day cookie, distinct from the per-user Calendar/Tasks OAuth scopes), how archive (`/archive`) and preferences (`/preferences`) pages query Firestore/Cloud Storage, and the actual implementation of `POST /api/survey/submit` (idempotency keying, async hand-off to `graph_update`).

### 3.5 Admin User Provisioning
PRD §3.3 step 1 says "Admin creates user record in Firestore" but doesn't say how — a script, an admin page, or a manual console edit. For 5–10 users this can probably be a deliberately minimal tool, but it should be a decision, not an afterthought, since it's the entry point for every other flow.

### 3.6 OAuth Token Storage & Refresh
Flagged as an open question (OQ-5B) in `cron-queue-worker.md` and never resolved: are tokens encrypted at rest in Firestore using a shared `shared/` utility, and is refresh handled per-worker or centrally? Needed before any worker that calls Calendar/Tasks/Gmail can actually be built.

### 3.7 Cloud Infrastructure & Deployment
`DEVELOPMENT.md` lists three GCP projects and "CI/CD tool TBD." Before implementation starts, this needs to firm up into: which CI/CD tool, how Cloud Run services/jobs, Cloud Scheduler jobs, and Cloud Tasks queues get provisioned (Terraform / gcloud scripts / console), and what the actual staging→prod promotion mechanism looks like.

### 3.8 Firestore Schema Consolidation & Isolation Enforcement
Collection schemas are currently scattered across the PRD and the two design docs. Before writing code, it's worth consolidating into one schema reference, and explicitly deciding how `user_id` data isolation is enforced — Firestore security rules, or purely application-level discipline (relevant since Cloud Run services likely use the Admin SDK, which bypasses security rules by default).

---

## 4. Explicitly not needed yet (Phase 2+)

To avoid over-designing ahead of need — these are real PRD features but scoped to Phase 2 or later, so they shouldn't block MVP design work:

- Evening capture / `evening-server` (Phase 2)
- Nudge-sender 8 PM flow (Phase 2)
- Shadow calendar full lifecycle — dedup badges, dismiss API, rolloff cron (Phase 2; the dedup *logic* is partially sketched in `cron-queue-worker.md`'s calendar-parse worker, which is fine to leave as-is)
- Weekly Rhythm Profile passive-observation bootstrap (Phase 2)
- Preferences page editing, selective graph injection, survey trend analysis (Phase 2/3)

---

*Daily Briefing System — Design Gap Tracker · April 2026 · Jack*
