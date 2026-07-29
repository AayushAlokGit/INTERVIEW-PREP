# Serval Technical Round Transcript
**Date:** 2026-05-29
**Start Time:** 7:17 PM
**End Time:** 8:35 PM
**Duration:** 78 minutes

## Part 1 — Coding
**Problem:** Insert Interval
**Topic:** Arrays / Intervals
**Difficulty:** Medium

### Problem Statement
Given an array of non-overlapping intervals sorted ascending by start, and a new interval, insert the new interval into the array such that intervals remain sorted and non-overlapping (merging overlapping intervals if necessary).

Example:
```
intervals = [[1,3], [6,9]], newInterval = [2,5]   → [[1,5], [6,9]]
intervals = [[1,2], [3,5], [6,7], [8,10], [12,16]], newInterval = [4,8] → [[1,2], [3,10], [12,16]]
```

Constraints:
- 0 <= intervals.length <= 10^4
- 0 <= start_i <= end_i <= 10^5
- Intervals are closed; sorted; non-overlapping in input

### Conversation Log

**Interviewer:** Presented the problem, asked for clarifying questions.

**Aayush:** "what are the constraints?"

**Interviewer:** Shared constraints and clarified intervals are closed.

**Aayush:** Initial approach — "two pass: pass 1 merge overlapping neighboring intervals of input, pass 2 modify intervals overlapping with newInterval, then merge."

**Interviewer:** Pushed back: input is guaranteed non-overlapping, so pass 1 is unnecessary. Asked to walk through example.

**Aayush:** Conceded pass 1 unnecessary. Described modifying overlapping intervals to expand to merged range, then merging at end.

**Interviewer:** Suggested cleaner one-pass: three phases — strictly-before, overlap (absorb into newInterval), strictly-after.

**Aayush:** Agreed, coded one-pass:
```cpp
#include <bits/stdc++.h>
using namespace std;
bool isOverlapping(vector<int> &a, vector<int> &b) {
    return !(a[1] < b[0] || a[0] > b[1]);
}
int main() {
    vector<vector<int>> intervals{{1,2}, {3,5}, {6,7}, {8,10}, {12,16}};
    vector<int> newInterval{4,9};
    vector<vector<int>> ans;
    int i = 0, n = intervals.size();
    while(i < n && intervals[i][1] < newInterval[0]) {
        ans.push_back(intervals[i]); i++;
    }
    while(i < n && isOverlapping(intervals[i], newInterval)) {
        newInterval[0] = min(newInterval[0], intervals[i][0]);
        newInterval[1] = max(newInterval[1], intervals[i][1]);
        i++;
    }
    ans.push_back(newInterval);
    while(i < n) { ans.push_back(intervals[i]); i++; }
}
```
Stated O(n) time, O(n) space.

**Interviewer:** Asked Aayush to trace edge cases (empty input, newInterval after all, newInterval before all, touching endpoint, fully contained).

**Aayush:** Asserted "yes it does handle all edge cases" without tracing.

**Interviewer:** Pushed: trace `intervals=[[1,5]], newInterval=[5,7]` explicitly.

**Aayush:** Traced correctly — phase 1 skipped, phase 2 merges to [1,7], phase 3 skipped, output [[1,7]].

**Interviewer:** Asked careful phrasing of space — O(n) output, O(1) auxiliary.

**Interviewer:** Asked for further optimization.

**Aayush:** Suggested binary search to find overlap region.

**Interviewer:** Probed — does it reduce overall asymptotic complexity?

**Aayush:** Correctly concluded O(n) is the floor due to output construction.

### Solution
**Aayush's Final Solution:** (see code above — one-pass three-phase)

**Time Complexity:** O(n)
**Space Complexity:** O(n) output, O(1) auxiliary

---

## Part 2 — Schema Design
**Scenario:** Event Ticketing System (Ticketmaster-like, assigned seating, 10-min seat holds)

### Conversation Log

**Interviewer:** Presented scenario, asked for core entities and query patterns.

**Aayush:** "User, Ticket, Event, Venue, Seat. Queries: events by date, tickets per event for admin, available seats for event."

**Interviewer:** Pushed — missed hold concept, missed user booking history.

**Aayush:** Added EventSeats junction; booking history via tickets owned by user.

**Interviewer:** Asked specifically about the hold mechanism — where state lives, expiration, race conditions.

**Aayush:** Proposed `EventSeatLock` with unique on (eventId, seatId) and expiresAt.

**Interviewer:** Asked about lifecycle — what happens on payment vs expiry; SQL for available seats.

**Aayush:** "On payment, lock deleted and ticket created. On expiry, background cleanup. Available = all seats - booked." Missed actively-locked-not-expired exclusion.

**Interviewer:** Pointed out locked-but-not-expired seats must also be excluded.

**Interviewer:** Asked for full table sketch.

**Aayush:** Initial sketch:
- User: id, name, email, metadata
- Venue: id, name, address, lat, lng, seats[] (denormalized)
- Seat: id, venueId, type
- Event: id, name, venueId, date, description
- EventSeats: id, eventId, seatId (unique on eventId+seatId)
- EventSeatLock: id, eventId, seatId, expiresAt (unique on eventId+seatId)
- **Ticket missing entirely**

**Interviewer:** Three pushbacks — missing Ticket, Venue.seats[] is unjustified denorm, EventSeats is redundant.

**Aayush:** Removed EventSeats, removed Venue.seats[], added Ticket: id, eventId, seatId, paymentDetails, createdAt.

**Interviewer:** Pointed out missing userId, missing uniqueness constraint, paymentDetails as column smell.

**Aayush:** Added userId, proposed `UNIQUE(eventId, seatId, userId)`, split Payment into its own entity.

**Interviewer:** Flagged critical bug — UNIQUE(eventId, seatId, userId) allows two users to both book seat A12.

**Aayush:** Corrected to UNIQUE(eventId, seatId).

**Interviewer:** Asked about indexes for the listed query patterns.

**Aayush:** "Event on date for browsing; Ticket on user for booking history and event for admin; EventSeatLock on expiresAt for cleanup, on event for admin."

**Interviewer:** Added that UNIQUE constraints provide indexes for free; suggested composite (date, venue_id) for richer browse filters.

### Final Schema

| Table | Columns | Constraints |
|---|---|---|
| User | id (PK), name, email, metadata | UNIQUE(email) |
| Venue | id (PK), name, address, lat, lng | |
| Seat | id (PK), venue_id (FK), row, number, type | UNIQUE(venue_id, row, number) |
| Event | id (PK), name, venue_id (FK), date, description | INDEX(date) |
| EventSeatLock | id (PK), event_id (FK), seat_id (FK), expires_at | UNIQUE(event_id, seat_id), INDEX(expires_at) |
| Ticket | id (PK), event_id (FK), seat_id (FK), user_id (FK), payment_id (FK), created_at | UNIQUE(event_id, seat_id), INDEX(user_id), INDEX(event_id) |
| Payment | id (PK), user_id (FK), amount, status, provider_txn_id, created_at | INDEX(user_id) |

### Key Design Decisions
- Drop EventSeats junction: every venue seat exists for every event; state is captured by Ticket (sold) and EventSeatLock (held). Available seats derived by anti-join.
- Lock and Ticket are separate tables with separate lifecycles. Payment success: delete lock, insert ticket. Expiry: background job purges where `expires_at < NOW()`.
- DB-enforced invariant: at most one Ticket per (event_id, seat_id) via UNIQUE constraint. Same on EventSeatLock.
- No denormalization of seats onto Venue — would create sync bugs with no clear read-side benefit.

---

## Feedback Given

**Coding scores:**
- Problem understanding & clarification: 3/5 — asked about constraints only when prompted, didn't proactively clarify closed-vs-open intervals.
- Approach & thought process: 3/5 — initial 2-pass missed the non-overlapping-input constraint; pivoted cleanly when pushed.
- Code quality & correctness: 4.5/5 — clean three-phase implementation with helper.
- Complexity analysis: 4/5 — correct O(n)/O(n); recognized binary search doesn't reduce floor.
- Communication: 3/5 — asserted edge cases without tracing until pushed.

**Coding avg: 3.5/5**

**Schema scores:**
- Entity identification: 3/5 — missed Booking-vs-Lock distinction, missed user history query pattern.
- Table design: 2.5/5 — forgot Ticket initially; unjustified Venue.seats[] denorm; redundant EventSeats.
- Indexing & query optimization: 3.5/5 — good coverage when prompted; didn't proactively note UNIQUE-gives-index-free.
- Normalization & tradeoffs: 3/5 — self-corrected denorm on challenge without articulating tradeoff.
- Edge cases & data integrity: 2/5 — critical UNIQUE(eventId, seatId, userId) bug would allow double-booking.
- Communication: 3.5/5 — receptive, iterates well, could defend choices more confidently.

**Schema avg: 2.9/5**

**Recurring themes:**
1. Read constraints carefully before committing — non-overlapping input on coding; user_id on Ticket in schema.
2. Trace edge cases, don't assert them.
3. Think hard about invariants the DB should enforce — walk through concurrent insert scenarios when writing UNIQUE constraints.
4. Don't denormalize without naming the read query it speeds up and the write-side cost.

**Time Taken: 78 minutes**
