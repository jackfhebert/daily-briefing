# Daily Briefing

A personal AI chief-of-staff that synthesizes calendar, tasks, forwarded emails, and weather into a daily morning briefing — plus a frictionless evening capture page so nothing gets lost overnight. Built for a small, invite-only group of users (5–10), not as a public product.

> **Status:** Pre-implementation. This repo currently holds planning docs only — no application code yet.

## What it does

- **Morning briefing** — a server-rendered page generated each day (~6 AM) that turns the day's calendar, open todos, parsed inbox items, and weather into a short, scannable digest with an AI-written summary.
- **Evening capture** — an always-available brain-dump page for offloading open loops and pending thoughts before bed; unresolved items resurface in the next morning's briefing.
- **Shadow calendar** — cross-checks dates extracted from forwarded emails (school newsletters, flight confirmations, etc.) against Google Calendar, flagging anything that's missing.
- **Knowledge Graph** — a growing set of markdown "node" documents (people, places, activities, preferences, weekly rhythm) that the system builds over time and injects into every briefing for context.
- **Preference learning** — lightweight per-section micro-surveys collect feedback that feeds back into the Knowledge Graph.

## Architecture at a glance

| Component | Trigger | Role |
|-----------|---------|------|
| `inbox-poller` / queue-worker | 5:00 AM | Parses new emails/tasks/calendar events into structured documents, updates shadow calendar |
| `briefing-generator` | 6:00 AM | Reads pre-parsed documents + Knowledge Graph, makes one Gemini call, renders the morning page |
| `web-server` | always on | Serves all pages (morning/evening/archive/preferences) and the survey API |
| `nudge-sender` | 8:00 PM | Emails a reminder if the user hasn't done their evening capture |

**Stack:** Google Cloud Run (jobs + service), Firestore (all data, namespaced per user), Cloud Storage (rendered HTML archive), Gemini 1.5 Pro (all LLM work), Google Calendar/Tasks/Gmail APIs, OpenWeatherMap.

See [`docs/OVERVIEW.md`](./docs/OVERVIEW.md) for a fuller architecture walkthrough, [`docs/PRD.md`](./docs/PRD.md) for the complete product requirements (data schemas, flows, phased rollout, open questions), and [`docs/DEVELOPMENT.md`](./docs/DEVELOPMENT.md) for the development process (tooling, environments, CI/CD, Claude Code workflow).

## Repo layout

```
docs/
  PRD.md          full product requirements document (source of truth)
  OVERVIEW.md     condensed architecture overview for orientation
  PROJECT.md      living status tracker: links, access, who has the project shared with them
  DEVELOPMENT.md  development process: tooling, environments, CI/CD, Claude Code workflow
  designs/        feature-level system design docs
    email-onboarding.md   email-first intro flow that seeds the Knowledge Graph
```

This repo is planned as a monorepo. Once implementation begins, top-level service directories will be added per [`docs/DEVELOPMENT.md`](./docs/DEVELOPMENT.md):

```
briefing-generator/   6 AM synthesis Cloud Run job
inbox-poller/         5 AM parse and queue worker
web-server/           serves briefing pages and survey endpoint
evening-server/       serves evening capture page
nudge-sender/         8 PM email nudge job
shared/               shared libraries, Firestore helpers, Gemini client
infra/                Cloud Run configs, IAM, scheduler definitions
```

## Status & roadmap

Build order follows the PRD's phased rollout:

1. **Phase 1 (MVP):** Cloud infra, OAuth onboarding, Knowledge Graph seeding, calendar + weather, morning briefing generator, survey endpoint.
2. **Phase 2 (Full V1):** Tasks, Gmail inbox parsing, shadow calendar, weekly rhythm, evening capture, nudges, preferences page.
3. **Phase 3 (Learning loop):** selective graph injection, profile change notifications, survey trend analysis.
4. **Phase 4 (V2):** email delivery, news/portfolio sections, agentic tasks.
