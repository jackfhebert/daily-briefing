# Daily Briefing System
## Web-Based Onboarding & Signup — System Design

**Status:** Design complete — ready for implementation planning
**Last Updated:** April 2026
**Author:** Jack

> **Relationship to the PRD:** this design elaborates on [PRD §3.3](../PRD.md#33-user-onboarding-flow) (User Onboarding Flow) and [PRD §3.4](../PRD.md#34-onboarding-self-introduction) (Onboarding Self-Introduction), incorporating the calendar-sharing model introduced in PRD v1.10 (no per-user OAuth in Phase 1 — see [PRD §5.1](../PRD.md#51-google-calendar)). It also resolves the relationship with [`email-onboarding.md`](./email-onboarding.md)'s email-first intro flow, which that doc explicitly left open (see §4 below).

---

## Overview

A new user is provisioned by the admin (Jack), then completes a short self-service onboarding flow at `/onboarding`: sign in with Google, share their calendar with the bot account, set a forwarding email address, and write a free-text self-introduction that seeds their Knowledge Graph. No step beyond admin provisioning + Google sign-in is required to "exist" in the system, and no step beyond sign-in is required to start receiving briefings — calendar sharing and the self-introduction are encouraged but not blocking, consistent with the system's general philosophy of degrading gracefully rather than gating on optional data.

This is a separate path into the system from the email-first intro flow (`email-onboarding.md`) — a user might do one, the other, or both, in either order. §4 defines exactly how the two reconcile.

---

## Components

### 1. Admin User Provisioning

**Tool:** a CLI script (`scripts/provision_user.py`), not an admin web page — for 5–10 invite-only users, a script run by Jack is simpler than building and securing an admin UI.

**Usage:** `python provision_user.py --email jack@example.com`

**Logic:**

1. Look up `users` collection by `email`
2. **No existing record:** create a new `users/{user_id}` document (`user_id` is a generated short ID, not derived from email) with `active: true`, `onboarding_complete: false`, `provisioning_source: "admin"`
3. **Existing record found (stub from email-intro flow):** the user already emailed `jhebert-bot@gmail.com` and has a stub record with partial Knowledge Graph data. Promote it in place — set `active: true`, `provisioning_source: "admin_promoted_stub"` — rather than creating a duplicate record. The existing `user_id` and Knowledge Graph nodes carry over untouched.
4. **Existing record found, already active:** no-op, print a message — re-running provisioning for an already-active user is harmless

**Why promotion instead of duplication matters:** the email-intro worker (`email-onboarding.md` §3 Step 1) already creates stub records keyed by email when an unrecognized sender emails the bot. If admin provisioning blindly created a second record for the same email, the user would end up with two `user_id`s and a split Knowledge Graph. Provisioning must always check for an existing record by email first.

---

### 2. Sign-In & Session (Web Server)

Google OAuth is used here strictly for **identity** — confirming "this Google account belongs to an invited user" — not for granting Calendar/Tasks API scopes. This is a narrower OAuth consent than a typical "Sign in with Google + access your data" flow.

**Flow:**

1. Unauthenticated request to any page → redirect to Google OAuth consent (`openid email profile` scopes only)
2. OAuth callback → look up `users` collection by the email returned in the OAuth profile
3. **No matching record:** render a static "You haven't been invited yet" page. The system never auto-creates a user from a bare sign-in attempt — provisioning (admin script or an email to the bot) must happen first.
4. **Matching record found:** issue a 30-day session cookie (per [PRD §14.1](../PRD.md#141-components)), set `user_id` in the session
5. If `onboarding_complete` is false → redirect to `/onboarding`; otherwise → `/dashboard`

All other pages and the survey endpoint trust the session cookie for `user_id` — never the request body — per the existing data-isolation principle in [PRD §3.1](../PRD.md#31-data-isolation).

---

### 3. Onboarding Page (`/onboarding`)

A single page with three sections, shown as a simple linear checklist rather than a strict wizard — sections can be completed in any order and revisited, since none of them block each other except the final "finish" action.

#### 3a. Calendar Sharing

- Static instructions: share your Google Calendar with `jhebert-bot@gmail.com`, permission level "See all event details"
- A **"Check connection"** button calls `POST /api/onboarding/check-calendar`, which attempts `calendar.events.list(calendarId: {user.email}, maxResults: 1)` using the bot's own credentials
- Result is written to the user record: `calendar_sharing_status: "confirmed" | "not_shared"`, `calendar_share_checked_at: <timestamp>` — and reflected immediately in the UI (✓ Connected / Not shared yet)
- This is the same field the 5 AM `calendar-parse` worker reads/writes opportunistically (per the `cron-queue-worker.md` §4b `not_shared` status) — onboarding and the daily worker share one source of truth, so a user who shares their calendar a week after onboarding doesn't need to come back to this page; the next 5 AM run picks it up automatically
- Not required to proceed

#### 3b. Forwarding Email

- Text input(s) for `forwarding_emails[]` — the address(es) the user will forward mail from
- Saved via `POST /api/onboarding/forwarding-email` on blur/submit — no verification step (no confirmation email sent); parsing simply starts whenever the first forwarded email arrives, matched by sender address per [PRD §3.2](../PRD.md#32-email-routing)
- Not required to proceed

#### 3c. Self-Introduction

- Same prompt as [PRD §3.4](../PRD.md#34-onboarding-self-introduction): "Introduce yourself as if you were meeting a new assistant who will be helping organize your day..."
- **If the user already has Knowledge Graph nodes** (from a prior email-intro thread — see §4), the page shows a short summary of what's already known ("Here's what we've picked up so far: ...") above the text box, framed as additive ("Anything to add or correct?") rather than starting from a blank form
- On submit, `POST /api/onboarding/introduction` makes a single synchronous Gemini call — same input/output contract as the email-intro worker's Gemini call (`email-onboarding.md` Gemini Output Schema: `profile_updates`, `clarifying_questions`, etc.), but with no `shadow_schedule_entries` extraction expectation at this stage and no `reply_warranted` field, since there's no email thread to reply into. Existing KG content (if any) is passed as the "existing context" input, the new free-text submission as "new content" — this reuses the exact prompt-construction pattern already designed for the email flow rather than inventing a second one
- Output is applied the same way: `create`/`update`/`none` per node, full-body replace on update
- Raw introduction text is retained on the user record per PRD §3.4 (appended to the root node until ≥5 nodes exist)
- Not required to proceed — a user can finish onboarding with an empty Knowledge Graph and let it build passively from calendar/email parsing in Phase 1, though the briefing will be noticeably less personalized until they write something

#### 3d. Finish

- A "Finish setup" button, visible regardless of which sections above are complete
- Sets `onboarding_complete: true`, redirects to `/dashboard`
- First briefing generated at the next scheduled 6 AM run, using whatever data sources are actually connected — degrades gracefully per the existing pattern (no calendar section if not shared, no email-derived content if no forwarding address was set, a thin Good-Morning summary if the Knowledge Graph is still empty)

---

## 4. Reconciling With the Email-First Intro Flow

`email-onboarding.md` deliberately left this open ("Deliberately Deferred" section). The rule:

- **Both flows write to the same `users/{user_id}` record, keyed by email.** Whichever flow touches a given email address first creates the record; the other flow finds and extends it rather than creating a second one.
- **Email-intro alone never activates a user.** A stub record created by the intro-worker has `active: false` (or simply isn't reachable, since admin provisioning is also the gate for sign-in — see below) until admin provisioning promotes it. This means a curious stranger emailing the bot can't get a live account; only an admin-provisioned email can ever reach `onboarding_complete: true` and start receiving briefings.
- **Sign-in is gated on provisioning, not on having a Knowledge Graph.** A user who has emailed the bot extensively but hasn't been admin-provisioned still can't sign in (§2 step 3) — the stub record exists, but isn't "invited" in the sign-in sense.
- **Order doesn't matter for the Knowledge Graph itself.** Whether a user's first KG-shaping input was an email or a self-introduction form, both write through the same `create`/`update`/`none` contract onto the same nodes — §3c explicitly surfaces existing KG content before asking for the web self-introduction so the two never silently overwrite each other.
- **After onboarding, both channels stay live.** A user can keep emailing the `jhebert-bot@gmail.com` "Intro" thread to add information at any time, even after completing web onboarding — `email-onboarding.md`'s `intro-poller`/`intro-worker` doesn't change behavior based on `onboarding_complete`.

---

## Firestore Fields Added to `users/{user_id}`

| Field | Type | Written by | Notes |
|---|---|---|---|
| `provisioning_source` | enum | provision_user.py | `admin` \| `admin_promoted_stub` |
| `calendar_sharing_status` | enum | onboarding page, calendar-parse worker | `unknown` \| `confirmed` \| `not_shared` |
| `calendar_share_checked_at` | timestamp | onboarding page, calendar-parse worker | Last time the status above was checked |
| `forwarding_emails[]` | string[] | onboarding page | Already specified in PRD §3.2; written here for the first time |
| `intro_text` | string | onboarding page (or intro-worker) | Raw self-introduction text, per PRD §3.4 |
| `onboarding_complete` | bool | onboarding page (`/finish`) | Already specified in PRD §3.3 |

---

## API Endpoints

| Route | Method | Purpose |
|---|---|---|
| `/onboarding` | GET | Renders the onboarding page; reads current state of all three sections |
| `/api/onboarding/check-calendar` | POST | Live-checks calendar sharing via bot credentials; writes `calendar_sharing_status` |
| `/api/onboarding/forwarding-email` | POST | Saves `forwarding_emails[]` |
| `/api/onboarding/introduction` | POST | Runs the Gemini self-intro call; applies `profile_updates` to the Knowledge Graph |
| `/api/onboarding/finish` | POST | Sets `onboarding_complete: true`; redirects client to `/dashboard` |

---

## Failure Modes

| Failure | Behavior |
|---|---|
| Admin runs provisioning script for an email with no Google account / typo | No detection possible at provisioning time — the record simply sits unused until someone signs in with a matching email. Not worth guarding against for a 5–10 user invite-only system. |
| `check-calendar` call fails (API error, not a "not shared" response) | Show a generic "couldn't check right now, try again" message; don't write `calendar_sharing_status` over a previously-confirmed state on a transient error |
| Gemini call fails during self-introduction submission | Show an error, let the user resubmit; nothing is partially written since the worker applies the full Gemini output atomically or not at all |
| User signs in before being provisioned | "You haven't been invited yet" page; no record created, no session issued |
| Stub record from email-intro never gets admin-provisioned | User can keep emailing the bot indefinitely, KG keeps growing, but they never receive a briefing — by design, this isn't a failure mode the system needs to detect or alert on |

---

## Deliberately Deferred

- **Email verification for forwarding addresses** — no confirmation email is sent when a user sets `forwarding_emails[]`; trusted-user assumption for a 5–10 person system. Revisit if the system ever opens beyond invite-only.
- **Self-serve sign-up** — there is no path for a stranger to provision themselves; admin provisioning is a hard gate. Out of scope by design (PRD §2.2 Non-Goals).
- **Admin web UI** — the provisioning script is intentionally a CLI tool, not a page. Revisit only if the user count grows enough that Jack running a script becomes a bottleneck.
- **Re-onboarding / un-completing onboarding** — there's no designed path to reset `onboarding_complete` back to false (e.g., if a user wants to redo the self-introduction from scratch). The Preferences page (Phase 2) already supports editing the introduction text per PRD §3.4; a full onboarding reset isn't needed on top of that.

---

*Daily Briefing System — Web-Based Onboarding & Signup Design · April 2026 · Confidential · Jack*
