# System Design Round Transcript
**Date:** 2026-06-08
**Start Time:** 10:27
**End Time:** 11:37
**Duration:** 70 minutes (~15 min spent on a skill-update detour; design itself ~55 min)
**Problem:** Design an Event Ticketing System (Ticketmaster)

---

## Conversation Log

**Interviewer:** Introduced the problem — event ticketing system handling a hot-event on-sale burst. Asked Aayush to gather requirements, functional scope first.

**Aayush:** Asked if 100M DAU scale is fine to assume.

**Interviewer:** Accepted but redirected — pin functional scope before numbers; noted the interesting number is the concentrated booking burst, not browse traffic.

**Aayush:** FRs — (1) search/view events, (2) view seats for an event, (3) book tickets for seats.

**Interviewer:** Scoped out payment internals and organizer event creation. Asked for NFRs with numbers: read:write ratio + QPS, consistency requirement on booking, and the hot-event burst magnitude.

**Aayush:** NFRs — (1) HA event querying 99.9% (~8.6 hrs/yr), (2) low latency p99 < 200ms, (3) strong consistency for booking (no double booking), (4) handle hot event via waiting room, 50k bookings/s, (5) 100M DAU × 5 queries/day = 500M/day, avg ~5k QPS, peak ~50k QPS.

**Interviewer:** Confirmed math. Reframed: 50k seats at "50k/s" sells out in 1 second — the real challenge is 1–2M users contending for 50k seats in the first seconds. Moved to core entities.

**Aayush:** Entities — User, Event(id, name, description, venueId, eventDate, artists[]), Venue(id, lat, long, name, description, address), VenueSeat(id, venueId, seatType, seatNumber), EventSeat(id, eventId, venueSeatId), Booking(id, userId, createdAt, eventId, venueSeats[], amount).

**Interviewer:** Praised VenueSeat/EventSeat split. Asked where seat availability state lives and whether available/booked is enough.

**Aayush:** Asked to keep entities as initial draft and evolve later.

**Interviewer:** Agreed, moved to API design.

**Aayush:** APIs (identity via JWT) — (1) GET /events?query=&cursor=&limit=&lat=&long=&radius= → Event[]; cursor pagination to survive dynamic event insertion. (2) GET /events/:id → Event + eventSeats[{id, seatType, status}]; added status (booked/available). (3) POST /events/:id/bookings, request {eventSeats[], amount}, idempotency key header for retry dedup.

**Interviewer:** Probed (1) client sending amount — trust boundary; (2) the flow gap — when does booking fire, and what protects the seat during the 2–5 min payment window?

**Aayush:** (1) amount computed server-side from eventSeat data. (2) Seat held ~10 min via a PUT endpoint that acquires locks by setting status=reserved with a TTL (Redis); user pays during hold; payment provider notifies via webhook; lock removed on completion.

**Interviewer:** Confirmed the three-state machine (available → reserved(TTL) → booked). Rendered API + updated entities. Asked for the HLD — read path at 50k QPS and write path under burst, with concrete datastores.

**Aayush (read path):** Client → API gateway (auth) → Query Service → Postgres for events. Postgres chosen for eventDate index, PostGIS spatial index on venue lat/long, and inverted indices for keyword search. Read replicas to distribute 50k QPS. Elasticsearch for efficient geo + inverted-index queries, kept consistent with DB via CDC events + workers. Redis cache for precomputed popular query results. Claimed no hot-key issue since 50k aggregate is fine for Redis.

**Interviewer:** Pushed back on hot-key reasoning — conflates aggregate throughput with per-key concentration; hot event = all users hitting same GET /events/:id key/shard. Asked for the write path.

**Aayush:** Corrected — for very popular queries, replicate the event list across multiple Redis nodes with client-side load balancing; accepts extra copies + eventual consistency.

**Aayush (write path):** Client → POST → Booking Service acquires Redis lock with TTL on chosen seats. Booking record created status=payment_pending (reconciliation jobs remove long-pending). Trigger payment provider with idempotency key = bookingId. On completion, webhook events pushed to Kafka for durability; workers consume, create/confirm booking, update event seats, delete lock. Seat update + booking status update in one atomic Postgres transaction (reason to store bookings in Postgres). Noted contention under hot event but Redis lock atomicity preserves strong consistency; high contention = degraded UX for losers.

**Interviewer:** Rendered full HLD. Deep dive 1 — the overselling race: A reserves seat, TTL expires, B books it, then A's late payment webhook arrives; two people booked. How does the worker prevent it? Deep dive 2 — build the waiting room.

**Aayush:** (1) Webhook fetches the booking's eventSeats, checks their availability/status; if already booked, trigger a refund event and log for debugging. (2) A queue in front of the booking service; when a user opens the booking page, their userId+eventSeats go into the queue and a connection is held; booking service consumes from queue head and notifies the user. Self-identified flaw: if A picks seat S and is ahead, and B later picks S and queues, B won't learn S is taken until reaching the head.

**Interviewer:** Endorsed the self-critique; noted the fix is to admit users to a live booking session (queue throttles concurrency, not seat assignment). On deep dive 1, noted the cleaner primary guard is a conditional atomic update (UPDATE ... WHERE status='reserved' AND reserved_by=A) making Postgres the arbiter, with refund as the rare exception. Final probe: name the actual Redis op for a single-seat lock, and how to make a 4-seat acquisition all-or-nothing.

**Aayush:** (2) Atomic Lua scripts for multi-seat all-or-nothing. (1) Not sure of the exact single-lock command.

**Interviewer:** Confirmed Lua answer correct; supplied SET seat:{id} {userId} NX EX 600 for the single-seat acquire. Wrapped to feedback.

---

## Design Summary

**Requirements Gathered:**
- FRs: search/view events, view event seats, book seats. (Out of scope: payment internals, organizer event creation.)
- NFRs: 99.9% availability for querying; p99 < 200ms read latency; strong consistency for booking (no double-book); hot-event handling via waiting room; 100M DAU × 5 q/day = 500M/day, ~5k avg / ~50k peak QPS; hot event 50k seats vs 1–2M concurrent users.

**High-Level Architecture:**
- Read: Client → API Gateway (auth) → Query Service → Postgres (PostGIS spatial, eventDate, inverted indices) + read replicas; Elasticsearch (geo + full-text) kept in sync via CDC + workers; Redis cache for popular queries, replicated across nodes for hot keys with client fan-out.
- Write: Client → API Gateway → Booking Service → Redis seat locks (reserve, TTL 10m, Lua for multi-seat atomicity); booking created payment_pending in Postgres; payment provider triggered with idempotency key=bookingId; webhook → Kafka (durable) → workers confirm booking + set seat booked + delete lock in atomic Postgres txn; reconciliation job expires stale pending bookings.

**Key Design Decisions & Trade-offs:**
- VenueSeat vs EventSeat separation (per-event sellable unit).
- EventSeat 3-state machine: available → reserved(TTL) → booked.
- Postgres for events (spatial + keyword + date indices) and bookings (atomic txn); ES for scalable geo/full-text reads (cost: CDC-based eventual consistency).
- Redis for seat locks (atomicity) and hot-query cache (replicated; cost: extra copies + eventual consistency).
- Kafka for webhook durability; idempotency keys for payment dedup; cursor pagination (survives dynamic insertion); server-side amount computation.

**Scalability & Fault Tolerance Points:**
- Read replicas + ES + replicated hot-key cache for 50k QPS reads.
- Redis lock atomicity for strong booking consistency; TTL prevents stranded holds.
- Reconciliation jobs for stuck pending bookings; refund compensation for overselling race; Kafka durability so webhook events aren't lost.

**Gaps / Missed Areas:**
- Could not name the single-seat lock command (SET NX EX); leaned on "Redis provides atomicity" as a phrase.
- Dual source of truth (Redis lock + Postgres status) — conditional-update arbiter only after prompting.
- Waiting room listed in NFRs but only designed when pushed; first cut coupled queue to seat selection.
- Postgres write hot-partition for a single hot event never raised.
- Monitoring/alerting (oversell detection, lock-leak metrics, queue-wait SLA) not addressed.
- Redis lock-node failure not discussed.
- Pace ran long.

---

## Feedback Given

### Standard Evaluation
- **Requirements:** Strong — FRs crisp; NFRs derived with unprompted BoE math; correctly reframed the burst (1–2M users vs 50k seats).
- **Core entities:** Strong — VenueSeat/EventSeat split is the key modeling insight, unprompted.
- **API design:** Strong — cursor pagination with reason, idempotency key, server-side amount, self-added status, self-designed reservation step.
- **HLD:** Strong — concrete datastores throughout (Postgres+PostGIS, read replicas, ES+CDC, Redis, Kafka); trade-offs named per component.
- **Scalability & fault tolerance:** Strong — Kafka durability, reconciliation jobs, TTL locks, hot-key fix all self-raised.
- **Deep dive:** Strong with one hand-wave — handled overselling race (refund), self-critiqued waiting-room flaw, nailed Lua multi-seat atomicity; couldn't name SET NX EX.
- **Communication:** Strong — reasoned aloud, self-corrected, volunteered own design flaws.

### Senior-Signal Scorecard
| Signal | Rating | Reason |
|---|---|---|
| Owns the narrative (self-raises traps) | Strong | Reservation TTL, idempotency, reconciliation, Kafka durability, hot-key — unprompted |
| Leads with trade-offs vs alternatives | Strong | ES vs Postgres, replicated cache copies vs consistency, each cost named |
| Pushes scale until it breaks | Mixed→Strong | Reframed burst, drove hot-key fix; waiting-room scale-break needed a pointer |
| API as a designed contract | Strong | Explicit shapes, cursor pagination + reason, idempotency, server-side amount |
| Operability / second-order concerns | Mixed | Reconciliation + refund logging good; missed monitoring, Redis-node failure, Postgres hot-partition |
| Pace | Mixed | High quality but ran long; comfortable phases could be tighter |

**Overall: Senior / Hire — bordering strong-hire on this problem.** Substance at the senior bar; held back by a few operability gaps not self-raised and one core mechanism not named precisely.

### What a senior strong-hire would have done on THIS problem
1. Named the lock op instantly (`SET seat:{id} {userId} NX EX 600`) and explained value=userId enables safe Lua compare-and-delete release.
2. Made Postgres the single arbiter from the start via conditional UPDATE, with Redis as optimistic fast-path — collapsing the dual-source-of-truth race unprompted.
3. Designed the waiting room as a concurrency throttle (Redis sorted set / token admission) admitting N users/sec to a live booking session, decoupled from seat choice.
4. Raised the Postgres write hot-partition for the single hot event and discussed per-event partitioning / in-memory seat-map authority.
5. Owned monitoring — oversell alerts, reservation-success rate, lock-expiry-vs-payment race metrics, queue-depth/wait SLA.

**Self-drill:** `system_design_senior_guidance.md`, items 5 (operability) and 6 (pace) — the two remaining gaps. Trap-raising and trade-off instincts are now at the senior bar.

**Time Taken: 70 minutes**
