# Project Overview

This document is a condensed map of the system for anyone getting oriented. For full detail and rationale, see [`PRD.md`](./PRD.md).

## What this is

Daily Briefing is a personal AI chief-of-staff for a small group of users (5–10, invite-only). Each day it produces:

- A **morning briefing** — an AI-synthesized digest of calendar, todos, forwarded emails, and weather, rendered as a permanent HTML page.
- An **evening capture** — a frictionless brain-dump page, available any time, that feeds open items back into the next morning's briefing.

The goal is to feel like a thoughtful assistant who already knows the user's priorities and family context, not a dashboard of raw feeds.

## Current status

Pre-implementation. This repository currently contains only planning docs (the PRD and this overview); no application code exists yet. Build order follows the phased rollout below, starting with Phase 1.

## Core concepts

**Knowledge Graph** — the single source of truth for everything the system knows about a user: a set of markdown "node" documents in Firestore (`users/{user_id}/knowledge_graph/`), rooted at an `[[About {Name}]]` node and branching into people, activities, places, projects, preferences, and a weekly rhythm node. Nodes are written and split automatically by Gemini as they grow. The full graph is injected into every synthesis call.

**Shadow Calendar** — a separate Firestore collection of dates extracted from forwarded emails (school newsletters, flight confirmations, etc.). At briefing time it's diffed against Google Calendar to either confirm an event or flag one that's missing from the calendar — the system's main safety net against missed events.

**Parsed Document Store + Task Queue** — all raw inputs (emails, tasks, calendar events, evening captures) are parsed by an LLM ahead of time into structured `parsed_documents`, via jobs in a Firestore-backed `analysis_queue`. The 6 AM briefing generator only ever reads pre-computed documents — it does no parsing itself, which keeps synthesis fast and debuggable.

**Multi-tenancy** — every Firestore path and Cloud Storage object is prefixed by `user_id`; every scheduled job loops over active users and processes each independently. There's no shared data between users, and email routing matches the `From:` header against known forwarding addresses.

## Daily cycle

| Time | Job | Does |
|------|-----|------|
| 5:00 AM | `inbox-poller` / queue-worker | Fetch new emails/tasks/calendar events, enqueue + process parse jobs, update shadow calendar, roll off expired events |
| 6:00 AM | `briefing-generator` | Read parsed documents + full knowledge graph, one Gemini synthesis call, render & store the morning HTML page |
| All day | `web-server` | Serve briefing/evening/archive/preferences pages, handle survey submissions |
| 8:00 PM | `nudge-sender` | Email a reminder to any user who hasn't opened the evening capture page yet that day |

Survey answers (per-section micro-surveys on the morning page, profile questions on the evening page) write into `survey_responses`, then asynchronously generate a short insight that's merged into the relevant Knowledge Graph node.

## System components

| Component | Role |
|-----------|------|
| `briefing-generator` | Cloud Run job, 6 AM trigger, one per active user |
| `inbox-poller` / `queue-worker` | Cloud Run job, 5 AM trigger; also drains the analysis queue |
| `web-server` | Cloud Run service serving all HTML pages + `POST /api/survey/submit` |
| `nudge-sender` | Cloud Run job, 8 PM trigger |
| Firestore | All user data, namespaced under `users/{user_id}/...` |
| Cloud Storage | Rendered briefing HTML, `briefings/{user_id}/YYYY-MM-DD/{morning\|evening}.html` |
| Secret Manager | Gemini API key; per-user OAuth tokens live encrypted in Firestore (NWS needs no key, just a `User-Agent` header) |

External integrations: Google Calendar API v3, Google Tasks API v1, Gmail API (read + send), National Weather Service API (per-user tracked cities, no API key required), and Gemini 1.5 Pro for all LLM work.

## Phased rollout (summary)

1. **Phase 1 — Foundation (MVP):** infra, OAuth onboarding, Knowledge Graph seeding, calendar + tracked-cities weather, Gmail inbox parsing + shadow calendar extraction/dedup, morning briefing generator, survey endpoint.
2. **Phase 2 — Full V1:** Tasks, shadow calendar confirmation badges (UI polish), weekly rhythm bootstrap, evening capture + nudges, full graph injection, preferences page.
3. **Phase 3 — Learning loop:** selective graph injection, profile change notifications, rhythm decay handling, survey trend analysis.
4. **Phase 4 — V2 features:** email delivery, news/portfolio sections, agentic tasks (flight search, research, bookings).

Full detail, schemas, and rationale for every section above live in [`docs/PRD.md`](./PRD.md).
