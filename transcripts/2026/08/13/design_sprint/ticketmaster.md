# Design Sprint Transcript (front half, timeboxed)
**Date:** 2026-08-13
**Start Time:** 12:04:42 · **End Time:** 12:20:55
**Problem:** Ticketmaster (ticket booking, high-contention on-sale)
**Front-half readiness: 2/5**

| Phase | Cutoff | Landed at | Cut off? | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 8:00 | 9:11 | Yes | 1 | 2/5 |
| Core entities | 12:00 | 9:11 | No | 1 | 3/5 |
| API design | 17:00 | 13:52 | No | 1 | 2/5 |

**First-pass completeness:** One message per phase, but each first pass was ~70% complete. Requirements omitted out-of-scope entirely, read:write ratio, storage growth, durability. Entities omitted the hold/reservation object and all keys/uniqueness constraints. API omitted idempotency, the hold endpoint, delete/cancel, and error semantics. Nothing was added by a second message because there was no second message — the gap is content density, not message count.

**Plausibility check:** Not performed. Asserted "1M booking requests/s" during a popular on-sale and never tested it. A one-line check — 1M/s × 60s = 60M requests against a venue holding ~50k seats — would have shown the number is off by orders of magnitude and reframed the contention problem as "50k seats, 500k concurrent users, one 10-second window" instead. Also derived "100M DAU × 10 events/day → 10k reads/s" without showing the divide (1B/86.4k ≈ 11.6k/s); the answer is roughly right but the arithmetic was never displayed or checked. Caps Requirements at 3.

## What he produced (verbatim, as it stood at each gate)

### Requirements
```
FRs ->
1. Users can browse and search for events.
2. Users can view seat availability at the venue for an event and book specific seats.

NFRs ->
1. Highly available for browsing events (99.9 ! 8.2 hrs downtime/year)
2. Eventually consistent for events
3. Low latency query for events (p99 < 200ms)
4. High consistency for seat booking , no double bookings.
5. Assuming 100M DAU , browsing 10 events/day -> 10k reads/s for events.
6. In popular times handle high contention for inventory upto 1M booking request/s
```

### Core entities
```
1. User (id, name)
2. Venue (id, name , location)
3. Seat(id, venueId, type)
4. Event (id, name, details, eventTime, venueId)
5. EventSeat(eventId, seatId, status)
6. Ticket(id, userId, price, eventSeatId, eventId)
```

### API design
```
1. GET /events?query={}&from={}&before={}&cursor={}&limit={}
Response: Event(id, name, details, date, venue:{Venue name, venue location})[]

2. GET /events/:id/eventSeatId/:eventSeatId
Response: EventSeat(status: AVAILABLE | BOOKED | RESERVED)

3. POST /events/:id/tickets ->
Request: {eventSeatIds[]}
Response: Ticket(id, date, eventSeatIds[], eventId)
```

## What was missing at each buzzer

**Requirements (cut at 8:00, submitted 9:11):**
- No explicit out-of-scope list at all.
- FRs thin: no seat *hold/reservation* flow, no payment, no cancellation/refund, no waiting room.
- No read:write ratio, no storage growth estimate, no durability posture.
- Consistency posture stated per subsystem (good) but availability target given only for browse, not for booking.
- Traffic model asserted, never sanity-checked; 1M booking req/s is implausible and unexamined.

**Core entities (inside gate):**
- **No hold/reservation entity** — the single object the contention problem requires.
- No `Booking`/`Order` separate from `Ticket`; no payment or idempotency record.
- No keys or uniqueness constraints (`EventSeat` needs unique `(eventId, seatId)`; no row identified as the lock target).
- No denormalised fields identified.
- `EventSeat` has `status` but no `holdExpiresAt` / `heldBy` / version column.

**API design (inside gate):**
- No idempotency key on `POST /tickets` — the most contended write in the system.
- No hold/reserve endpoint, so the API cannot express the booking flow.
- No seat-map list endpoint (only a single-seat lookup); no DELETE/cancel.
- Response shapes are type sketches, not contracts: no envelope, no `nextCursor`, `details` is not a field.
- Cursor-vs-offset never justified; pagination present only on `/events`.
- No error/status semantics (409 seat taken, 410 hold expired, 402 payment).
- Price absent from the booking request/response.

## Where the time went
Requirements over-ran by 1:11. The cost was in the two derived numbers — 10k reads/s and 1M bookings/s — both of which were narrated rather than stated, and neither of which survived scrutiny. Recall-level NFRs (99.9%, p99 <200ms) were written out longhand when they are four-second statements. Entities and API were both comfortably inside their gates; API finished 3:08 early and that spare time was not spent adding shapes, idempotency, or the missing endpoints.

## Feedback given
Front half would have reached the HLD at ~minute 21 of a 45-minute round carrying no out-of-scope list, no hold entity, and a booking endpoint with no idempotency — the deep dive would have gone to repairing the contract rather than the contention algorithm. The design instincts (Seat vs EventSeat split, consistency-per-subsystem) are sound; the front half is losing on completeness and on unchecked numbers, not on understanding.
