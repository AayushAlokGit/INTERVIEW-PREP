# System Design Round Transcript
**Date:** 2026-06-22
**Start Time:** 11:46
**End Time:** 12:15
**Duration:** 29 minutes (round ended early by candidate, at the API stage)
**Problem:** Design Google Calendar (Calendaring & Scheduling System)

---

## Conversation Log

**Interviewer:** Presented the open-ended prompt: design a calendaring/scheduling system like Google Calendar (create events, reminders, invites, day/week/month views, free/busy). Asked Aayush to start with requirements gathering and drive the scoping. Set up draw.io canvas; interviewer owns all diagramming.

**Aayush:** Noted start time 11:46. Asked: do we need to support recurring events?

**Interviewer:** Yes — recurring events in scope. Asked him to drive the FRs and NFRs with numbers.

**Aayush:** Asked what scale to assume.

**Interviewer:** Gave one anchor — 1B registered users, 300M DAU — and asked him to derive reads/writes/storage himself.

**Aayush:** FRs: (1) create dated events + reminder config, (2) invite others, (3) view personal calendar day/week/month with schedule, (4) view others' calendars for availability, (5) recurring events. NFRs: HA 99.99% (~52 min/yr), p99 < 200ms render, consistent view of others' calendars (avoid double-booking). BoE: 300M DAU × 3 weekly views/day, each view ≈ 10 event reads, 1 event/day per user → stated ~30k reads/s and ~3k writes/s, peak 5×.

**Interviewer:** Flagged the read arithmetic: 300M × 3 = 900M views/day ≈ 10.4k views/s → ×10 = ~104k event reads/s, not 30k. Asked him to recheck and confirm read/write ratio.

**Aayush:** Conceded the calculation slip.

**Interviewer:** Locked anchors — reads ~104k/s avg, ~520k/s peak; writes ~3.5k/s avg, ~17k/s peak; ~30:1 read-heavy. Rendered FRs/NFRs to canvas. Asked for core entities before architecture, flagging how recurrence and invitations affect the model.

**Aayush:** Entities v1: CalendarEvent (id, calendarId, date, attendees, organiser, name, description, recurrenceSchedule); User (id, name, email); Calendar (id, userId).

**Interviewer:** Pushed three gaps: (1) attendee RSVP state, (2) where reminder config lives + per-attendee reminders, (3) recurrence — one row or 260 rows for "standup every weekday for a year"?

**Aayush:** Entities v2: CalendarEvent (id, calendarId, nextOccurrenceDate, attendees, organiser, name, description, recurrenceSchedule, reminderSchedule); User; Calendar; EventInvite (id, fromUserId, toUserId, status, eventId).

**Interviewer:** Approved EventInvite + status and the one-row-recurrence-rule choice. Noted reminderSchedule on the event makes reminders shared, but they're personal (park it). Rendered entities to canvas. Asked for the full API contract: create (recurring + invitees), fetch calendar by range, RSVP, free/busy — with verbs, request/response shapes, pagination, idempotency.

**Aayush:** APIs: identity from JWT. POST /events with Idempotency-Key header, req {eventName, eventAttendees, recurrenceSchedule, reminderSchedule}, resp = event details + invite status. GET /calendars?timeSlice={3days|7days|30days}&userId={}, resp = event details; userId in URL to view another user's calendar (accepted trade-off).

**Interviewer:** Pushed: response shapes too vague (asked for explicit fields + how a recurring event appears in the response — one rule object or expanded instances, expanded by whom); pagination on GET /calendars (cursor vs offset); two missing endpoints (RSVP, and free/busy as a separate detail-free contract vs full calendar view + privacy).

**Aayush:** Asked to end the session and have the interviewer complete the calendar design.

**Interviewer:** Ended the round. Delivered observed-portion feedback, senior-signal scorecard, and a full reference design (completed API, HLD, recurrence + exceptions, scale-break, reminders at scale, consistency, named trade-offs, operability). Asked for end time.

**Aayush:** Noted end time 12:15.

---

## Design Summary
**Requirements Gathered:** FRs — create events + reminders, invite others, personal calendar views (day/week/month), view others' availability, recurring events. NFRs — HA 99.99%, p99 < 200ms render, consistent free/busy view; 1B users / 300M DAU; reads ~104k/s avg & ~520k/s peak; writes ~3.5k/s avg & ~17k/s peak; ~30:1 read-heavy (read arithmetic corrected after a 3× slip).

**Core Entities:** CalendarEvent (single row + recurrenceSchedule/RRULE, nextOccurrenceDate, reminderSchedule), User, Calendar, EventInvite (per-attendee RSVP status).

**High-Level Architecture:** Not reached — round ended at the API stage. No components drawn by candidate.

**Key Design Decisions & Trade-offs:** Idempotency-Key on create (self-raised); one-row recurrence-rule model over materializing instances; userId-in-URL trade-off acknowledged.

**Scalability & Fault Tolerance Points:** Not reached.

**Gaps / Missed Areas:** Vague response shapes (no explicit fields); no pagination on calendar fetch; RSVP and free/busy endpoints undefined; free/busy-vs-full-view privacy distinction not made; recurrence expansion semantics in the read path not addressed; no HLD, no deep dive, no scale-break, no operability. Reminder ownership (per-user vs per-event) only flagged by interviewer. Asked interviewer to derive scale numbers and to complete the design.

---

## Feedback Given

**Requirements (good):** Clean FRs, recurring events correctly scoped in. NFRs with numbers and a read/write ratio. The 3× read-arithmetic slip is a recurring pattern — method right, multiplication wrong; sanity-check the unit (views/s vs event-reads/s).

**Entities (good, after prompting):** First pass embedded attendees and had no reminder/RSVP modeling. After a nudge, landed on EventInvite with status and the one-row + recurrence-rule model — both correct senior calls, but needed prompting rather than self-raising them.

**API (incomplete):** Idempotency key on create was a strong self-raised move. But response shapes stayed vague, pagination unaddressed, and the round ended before RSVP and free/busy — contract ~40% done.

**Never reached HLD, deep dive, or scale-break** — the level-differentiating part.

### Senior-Signal Scorecard (observed)
| Signal | Status | Reason |
|---|---|---|
| Owns the narrative / self-raises traps | Mixed | Self-raised idempotency; prompted for RSVP state, reminder ownership, recurrence materialization |
| Leads with trade-offs vs named alternatives | Mixed | Only the idempotency/userId trade-offs shown; round ended before architecture |
| Pushes scale until it breaks | Mixed | Never reached HLD/scale discussion |
| API as a designed contract | Mixed | Idempotency good; vague response shapes, no pagination, two endpoints missing |
| Operability / second-order | Weak | Not reached |
| Pace | Weak | Ended the round at the API stage; core design never completed |

**Level read:** Insufficient signal to assess senior — round stopped early. Pattern matches history: correct calls, but led to them, and ended before the hard part. Biggest lever: pace + reaching the deep dive (requirements/entities/API done by ~20 min so the back half is the hard problem).

### What a senior strong-hire would have done on THIS problem
- **Self-raised the traps:** RSVP needs its own state; reminders are per-user not per-event; recurrence must be one RRULE row + bounded expansion, not 260 rows; free/busy is a separate, detail-free contract for privacy.
- **Finished the API as a contract:** explicit response fields, cursor pagination (live-changing data), server-side recurrence expansion, PUT /rsvp, GET /freebusy returning busy blocks only.
- **Reached the hard cores:** recurrence + single-instance exceptions/overrides + "this-and-following" split (UNTIL + new event); scale-break on "find a slot for 10 people" / giant shared calendars (hot key → materialized per-user busy intervals, sharded, unioned); reminders at scale via bucketed due-index + queue + idempotent dispatch + lag monitoring; resource-booking conflict via conditional write + optimistic concurrency.
- **Operability:** cache hit ratio, expansion cost caps, hot-calendar detection, reminder-lag alerting, fleet cost at 520k reads/s.

**Time Taken: 29 minutes** (round ended early by candidate)
