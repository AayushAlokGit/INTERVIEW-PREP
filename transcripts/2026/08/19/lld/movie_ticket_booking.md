# LLD Round Transcript
**Date:** 2026-08-19
**Start Time:** 15:31:52 · **End Time:** 16:21:23 · **Duration:** 50 min (round content complete at 16:19:12 = 47 min)
**Problem:** Movie Ticket Booking with Seat Holds
**Category:** Resource allocation
**Difficulty:** 4/5 — per-show seat state must be separated from physical seats, an atomic multi-seat hold with time-based expiry, and a seat-contention concurrency story that is the whole point of the problem
**Performance Rating:** 2/5  <!-- machine-read on future rounds; ≤2 = eligible for re-ask, ≥3 retired -->
**Hints Used:** 0/2
**Requirements Asked:** the operations list, the reserve/book rules, the theatre/screen/show structure · **Never Asked:** whether passed-in seats belong to the show, whether the seat list can contain duplicates, cap on seats per booking, what happens to a hold once the show starts, **any concurrency posture**
**Walk coverage:** 3/8 Hit, 4 Partial, 1 Miss
**Out of Scope list produced:** Unprompted
**Self-Verified:** No — no trace offered; when given two scenarios, got the first right and **the second wrong**, and the second contained the bug
**Concurrency follow-up:** two threads racing `reserveSeats` on one seat — identified the interleaving, went coarse vs fine-grained locking, **stated the throughput cost unprompted**, then nailed lock-ordering deadlock on the follow-up. Named the mechanism ("race condition") rather than the category (lost update / check-then-act correctness).
**Would it have fit a real 40-min round?** No — cut off mid-extensibility, around the seat-tier follow-up. The concurrency discussion (his best work) would have been lost.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements + Out of Scope | 5 min | 7 min | +2 | Over |
| Entities & relationships | 8 min | 13 min | +5 | Over |
| Class design | 20 min | 26 min | +6 | Over |
| Implementation + trace | 32 min | 36 min | +4 | Over |
| Extensibility + concurrency | 40 min | 47 min | +7 | Over |
| **Total** | 40 min | **47 min** | **+7** | Over |

Stamps: start 15:31:52 · Q1 15:33:07 (+1m) · Q2 15:35:02 (+3m) · Q3 15:37:19 (+5m) · **requirements + OOS 15:38:30 (+7m)** · **entities 15:44:53 (+13m)** · entity probes answered 15:45:50 (+14m) · **class design 15:58:09 (+26m)** · class probes answered 16:00:07 (+28m) · implementation described in prose 16:04:27 (+33m) · **pseudo-code submitted 16:07:35 (+36m)** · traces answered 16:11:41 (+40m) · extensibility answered 16:13:47 (+42m) · concurrency answered 16:17:47 (+46m) · **lock-ordering answered 16:19:12 (+47m)** · feedback 16:21:23

---

## Problem Statement

> Low-level design round. Single process, in-memory — I want the object model, not infrastructure.
>
> **Design a movie ticket booking system with seat holds.**
>
> **Difficulty: 4/5**

Five phases and the reference timeline stated up front; clock measured, not enforced. Language: pseudo-code. No UML.

---

## Conversation Log

**Interviewer:** [presents problem, difficulty, five phases, reference timeline, design sheet path] Which will you write in for phase 4 — pseudo-code, C#, Python, or Java? Any questions before you start?

**Aayush:** pseudocode for phase 4. 1. What are the operations the system must support?

**Interviewer:** A user browses shows for a movie, picks seats for a show, holds them while they pay, and completes the booking. That's the product intent — the rest of the shape is yours to define.

**Aayush:** what are the rules for reserving and booking ticket. Assuming that a seat for show will be either available, reserved or booked

**Interviewer:** Yes, those three states are right. A hold has a 5-minute expiry; when it expires the seats return to available with no user action. A user can only book seats they currently hold, and only their own hold. A hold on multiple seats is all-or-nothing — if any requested seat isn't available, the whole hold fails and nothing is held. Anything else is yours to decide.

**Aayush:** how many theatres will be managed in the system and how will the movie shows be set in each theatre?

**Interviewer:** Many theatres, each with multiple screens. A screen has a fixed seat layout. A show is one movie playing on one screen at one start time — assume shows are already created by an admin path you don't need to design.

**Aayush:** Requirements ->
1. Actors -> Users, BookingSystem
2. Operations -> Users browse shows for a movie, users check available seats, they can reserve seats for some time period before checkout, users can book the tickets for a movie show.
3. Rules -> Seat can be held for 5 minutes. Seats become available when holds expire, user can only book the seats they currently hold and only their held seats. Holds are acquired atomically over all requested seats.
4. Seats for a show have 3 states AVAILABLE, RESERVED, BOOKED
5. There are multiple theatres, each having multiple screens (with fixed layout), different movies play on the screen at different times.

Out of scope ->
1. UI
2. Movie show addition.

**Interviewer:** Requirements and Out of Scope are on the sheet. Move to entities and relationships.

**Aayush:** [entity list — see Design section, including `Booking(..., amount, ...)` and `BookingSystem(Map<string, List<MovieShow>>)`]

**Interviewer:** Two questions. (1) `BookingSystem(Map<string, List<MovieShow>>)` — what's the key? (2) `Booking.amount` — which requirement needs it?

**Aayush:** 1. movie name is the key. 2. No requirement needs it so it can be ignored for this design as payment is out of scope of this design

**Interviewer:** Noted — `amount` dropped, and I'll fold payment into out of scope. Move to class design: state and method signatures.

**Aayush:** [class outlines for BookingSystem, ShowSeat, MovieShow — see Design section]

**Interviewer:** Two things. (1) `BookingSystem.bookSeats(movieShow, showSeats) -> Booking` takes no user, but your rule says a user can only book seats they hold, and `ShowSeat.bookSeat(user)` needs one. Reconcile that. (2) A hold is taken at 10:00 with a 5-minute expiry. Nobody touches the system until 10:07, when a different user calls `getAvailableSeats`. Walk me through what those seats' `status` is at that moment, and who changed it.

**Aayush:** 1. need to add user param to BookingSystem.bookSeats. 2. The status of a held seat changes only on reads so the seats will be in reserved status and will remain to be in reserved state until read at a point after 10:05

**Interviewer:** Lazy expiry — fine. Write the implementations for the two methods that carry the logic: the hold path and the book path.

**Aayush:** BookingSystem.reserveSeats -> Check the current status of the given show seats if any of them not available then throw exception. holdExpiryTime = now + holdExpiryTimeout. Call the movieShow.holdSeats(user, List<ShowSeat>, holdExpiryTime) to change status of all seats to reserved.
BookingSystem.bookSeats -> Check that the user initiating the booking request has acquired an unexpired hold on the seats. If no then throw exception. Call movieShow.bookSeats to change seat status and generate booking object

**Interviewer:** That's the description — I need the code. Write it as pseudo-code, including the `MovieShow` and `ShowSeat` methods these call.

**Aayush:** [full pseudo-code for ShowSeat.holdSeat / freeSeat / bookSeat and MovieShow.holdSeat / bookSeat — see Design section]

**Interviewer:** Two scenarios — walk each through your code and tell me the resulting state. (1) User A holds `[S1, S2]` at 10:00 (5-min expiry). At 10:06, User B calls `MovieShow.holdSeat(B, [S2, S3], now+5min)`. What is the status and `holdAcquiredBy` of S1, S2 and S3 after that call returns? (2) `MovieShow.holdSeat(A, [S1, S1], now+5min)` — same seat twice in the list. What is the state of S1 after the call returns?

**Aayush:** 1. s1 still held by A it will be unheld when someone tries to read it. s2 and s3 held by B until 10:11 after that any read of these seats will remove and free the seats.
2. no different the hold will always be acquired by same user and for same expiry time for the seat just wasted iterations

**Interviewer:** Noted. Two extensions: (1) Seats now have tiers — standard, premium, recliner — priced differently, and a show can run a promotion that discounts one tier. Where does that land in your design? (2) A user can cancel a booking up to 30 minutes before showtime; the seats return to the pool. Where does that land?

**Aayush:** 1. showSeat will have the tier information for the seat for that particular show. And promotions are also a show level details so should be present in movieShow. 2. another method cancelBooking to be added to BookingSystem, MovieShow

**Interviewer:** Last one. Two users call `reserveSeats` for seat `S1` at the same instant, on different threads. Walk me through what breaks in your code, and what you'd do about it.

**Aayush:** update the md file with the class design it is missing there

**Interviewer:** [appends sections 4–6 to the sheet] Back to the open question: two users call `reserveSeats` for seat `S1` at the same instant, on different threads. What breaks in your code, and what would you do about it?

**Aayush:** both the threads could see S1 as available and then who will be able to reserve S1 will depend on the order of execution — a race condition. This issue can be solved by introducing a lock on the seat map of a movie show and using locks. A coarse lock would reduce the throughput of reservation and booking; we could have fine grained locks for each seat in the show, so that reservation and booking throughput remains high.

**Interviewer:** Your hold is atomic across a list of seats. With per-seat locks, what does that require, and what new risk does it introduce?

**Aayush:** the order of acquiring the locks becomes important. If the order for acquiring locks is not consistent for a given show seat then there are chances of there being deadlock. To eliminate deadlock chances the locks for the seats must always be taken in a consistent order.

**Interviewer:** Correct — consistent lock ordering, typically by seat id. That's the round. [feedback follows]

---

## Walk coverage (phase 1)

| # | Item | Hit/Partial/Miss | Evidence |
|---|---|---|---|
| 1 | Actors & entry point | **Hit** | *"Actors -> Users, BookingSystem"* — orchestrator named in the first line |
| 2 | Core operations | **Partial** | *"What are the operations the system must support?"* — asked rather than proposed, then restated the interviewer's answer without adding one of his own |
| 3 | Rules & legality | **Partial** | *"what are the rules for reserving and booking"* — same pattern. One rule of his own arrived later (`InvalidShow if time in past`) but in a signature, not in phase 1 |
| 4 | Lifecycle & terminal states | **Hit** | *"Assuming that a seat for show will be either available, reserved or booked"* — proposed the state machine himself, unprompted. Best moment of phase 1 |
| 5 | Failure behaviour | **Partial** | Never raised as a requirement, but the convention was held perfectly in the class design — exceptions throughout, seven named specifically, consistent across every signature |
| 6 | Multiplicity & domain variants | **Partial** | Asked about theatres/screens/shows and got the structure right; never asked about seat types despite putting `type` on `Seat`, never about pricing or per-user limits |
| 7 | Concurrency posture | **Miss** | Never mentioned in phase 1, on a seat-contention problem. Handled well at minute 46, but only because the interviewer served it |
| 8 | Explicit Out of Scope | **Hit** | Produced unprompted; extended to payment during the `amount` probe |

**3 Hit · 4 Partial · 1 Miss** — his strongest walk in this file's history.

**Dangling rules:**
- *Booking cancellation up to 30 min before showtime* — answered "add `cancelBooking` to BookingSystem, MovieShow" and stopped. `Booking` has no state field, so a cancelled booking is indistinguishable from a live one, and nothing owns the 30-minute check.
- *`Seat.type`* — on the entity from minute 13 enforcing nothing. The tier follow-up retroactively justified it, but at the time it was the same unjustified state as `amount`; he only dropped the one that was pointed at.

**Silent assumptions:**
| Assumption | What it cost |
|---|---|
| Passed-in `ShowSeat` objects belong to *this* show | Nothing validates it — seats from show A can be passed to show B's `holdSeat` and will be held |
| The seat list contains no duplicates | **This is the bug** — see below |
| `getAvailableSeats` → caller picks → caller calls `reserveSeats` is safe | Live references handed out then passed back; a check-to-act window even single-threaded, never named |
| No cap on seats per booking | One user can hold an entire screen |

---

## His Design

**Requirements he gathered:** Actors (Users, BookingSystem); operations (browse shows, check available seats, reserve for a period, book); rules (5-min hold, auto-free on expiry, book only your own held seats, atomic multi-seat hold); seat states AVAILABLE/RESERVED/BOOKED; multi-theatre / multi-screen / fixed layout / shows at times.

**Out of Scope:** UI, movie show addition (payment folded in during probing).

**Entities & relationships:**
```
Movie(name)
Theatre(List<Screen>)
Screen(List<Seat>)
Seat(id, type)
MovieShow(time, movie, screen, theatre, List<ShowSeat>)
ShowSeat(seat, status, holdAcquiredBy, holdExpiryTime)
Booking(movieShow, seats, createdAt, user)          // `amount` dropped on probe
BookingSystem(Map<string movieName, List<MovieShow>>)
enum SeatStatus: AVAILABLE | RESERVED | BOOKED
```

**Class outlines:** see `movie_ticket_booking_design.md` §4 — full state and signatures for `BookingSystem`, `ShowSeat`, `MovieShow`, plus the `user` parameter added to `bookSeats` under probing.

**Core implementation:** see `movie_ticket_booking_design.md` §5 — `ShowSeat.holdSeat / freeSeat / bookSeat` and `MovieShow.holdSeat / bookSeat` in full pseudo-code.

**Gaps / misplaced responsibilities:**
- **The bug (never caught).** `MovieShow.holdSeat(A, [S1, S1], T)`: check loop passes (S1 AVAILABLE, checked twice); set loop's first call sets `status = RESERVED, holdExpiryTime = T`; the **second** call reads `status == RESERVED` and `currentTime < holdExpiryTime` and **throws `SeatAlreadyReserved`** mid-mutation. With `[S1, S2, S1]` it leaves S1 and S2 held while the caller sees an exception — the exact atomicity guarantee the check-then-set structure exists to provide, broken by the structure itself.
  - **He claimed:** *"no difference … just wasted iterations."*
  - **Root cause is architectural:** `ShowSeat.holdSeat` re-validates and `MovieShow.holdSeat` validates the same conditions beforehand — two copies of one rule, disagreeing about a state the loop itself creates.
- `MovieShow.holdSeat` reaches into `seat.status` and `seat.holdExpiryTime` to make a decision the seat should make (Ask, not Tell) — and `BookingSystem.reserveSeats` checks it a third time.
- `Booking` has no state — exists only to be returned, never queried or transitioned; cancellation cannot work without it.
- No validation that supplied seats belong to the show.
- `MovieShow.bookSeat`'s validation loop throws `SeatHoldExpired` without freeing, while `ShowSeat.bookSeat` frees before throwing — inconsistent between the two layers.

---

## Feedback Given

### Round conditions
- **Hints used: 0/2.** No ceiling from hints.
- **Requirements asked:** operations list, reserve/book rules, theatre/screen/show structure — three well-formed questions.
- **Never asked:** whether seats passed in belong to the show, whether a seat list can contain duplicates, cap on seats per booking, what happens to a hold once the show starts, and — the standing one — **any concurrency posture**, on a problem whose entire reason for existing is seat contention.
- **Out of Scope: produced unprompted**, and extended when the `amount` probe landed. Thin, but there without a reminder.
- **Self-verified: No.** Submitted and stopped. Given two scenarios, got the first right and the second wrong — and the second contained the bug.

### Rubric

**Requirements & scoping — 3/5.** Out of Scope unprompted, lifecycle self-proposed, three well-aimed structural questions. But operations and rules were extracted from the interviewer rather than proposed and confirmed — the difference between scoping and interviewing your interviewer. Concurrency posture missed.

**Entity modelling — 5/5.** `ShowSeat` is the entire problem and he got it in his first entity list with no prompting. Separating the immutable physical `Seat` (belongs to a `Screen`) from the per-show `ShowSeat` carrying status, holder and expiry is the modelling decision this problem tests; most candidates conflate them and discover at implementation time that seat state is per-show. Everything downstream was clean because this was right.

**Class design — 4/5.** Specific, well-named exceptions across every signature — seven, all distinct and meaningful. Clear three-layer split. Docked for `bookSeats` missing the `user` its own rule requires (fixed on probe, but it shipped), and for `Booking` having no state.

**Responsibility placement — 3/5.** Good at the leaf: `ShowSeat` owns its transitions, `freeSeat()` and `bookSeat()` are behaviours not setters, nothing outside sets `status`. Broken one layer up: `MovieShow.holdSeat` reaches into `seat.status`/`seat.holdExpiryTime`, and `BookingSystem.reserveSeats` checks the same thing a third time. Three copies of one rule is what produced the bug.

**Implementation & correctness — 2/5.** Structure is right — check-all-then-mutate-all is the correct shape, and the expired-hold-then-reacquire path inside `ShowSeat.holdSeat` is genuinely well handled. But the atomicity it exists to guarantee doesn't hold, and he asserted it did when asked directly.

**Simplicity & judgement — 5/5.** `Booking.amount` challenged, examined, dropped in one line — no defending, no "we might need it later". Zero patterns forced. Consistently his strongest dimension.

**Extensibility — 3/5.** Both follow-ups landed on plausible seams — tier on `ShowSeat` (per-show, so a premium seat can differ by show), promotion on `MovieShow` (show-level scope). Correct instincts, but both one-line answers, and cancellation genuinely doesn't work without a `BookingStatus` he didn't add.

**Concurrency — 4/5.** Strongest phase. Identified the interleaving, went straight to coarse vs fine-grained, **stated the cost unprompted** (throughput on reservation and booking) — historically the missing half — then nailed lock ordering on the follow-up. Docked only for naming the mechanism ("a race condition") rather than the category: this is a *correctness* failure, a lost update on a check-then-act.

**Communication — 4/5.** Clear, responsive, and **no long silences** — longest gap was 12 minutes spent producing a full class design. One turn spent asking for a sheet update when the content was already there.

### Pace report
[table as above] **Would this have fit a real 40-minute round? Just barely not.** The cut lands **mid-extensibility, around the seat-tier follow-up.** He'd have delivered requirements, entities, a complete class design and working implementations — a real deliverable. What's lost is the concurrency discussion, his best work and the single thing that most separates a senior read from a mid one on a seat-booking problem.

Seventeen percent over, no phase over by more than six minutes — steady drag, not a blowout. Biggest single sink is class design at 12 minutes of composition, much of it re-typing the entity list already given in phase 2 — the third consecutive round with that pattern.

This was a 4/5 problem. **The overrun is process, not size.**

### **Performance Rating: 2/5**

**This would have been a 4, capped at 2 — core logic with a flaw he never caught.** The ceiling is not a comment on the design, which is good: the entity model is the best in this file, the exception discipline is genuinely senior, and the concurrency answer would pass at most bars. It's a comment on shipping. He wrote a method whose only job is atomicity, was handed the exact input that breaks it, and confirmed it was fine.

### Senior-signal scorecard

| Signal | Status | Why |
|---|---|---|
| Scopes before designing | **Mixed** | Out of Scope unprompted and the seat state machine self-proposed — both real. But operations and rules were requested from the interviewer rather than proposed for confirmation, and concurrency posture was never established. |
| State derived from requirements | **Strong** | `ShowSeat`'s fields map one-to-one onto the hold rules — nothing speculative. `amount` dropped the moment it couldn't be tied to a requirement. |
| Rules live with their state (Tell, Don't Ask) | **Mixed** | Excellent at the leaf: `ShowSeat` owns every transition, no external setters. Broken at the aggregate: `MovieShow` and `BookingSystem` both re-implement checks the seat already owns. |
| Simplicity held under pressure | **Strong** | No patterns performed; speculative state removed on challenge without defending it. |
| Verifies own logic | **Weak** | No trace offered unprompted; when prompted, the one scenario carrying the bug was answered wrong. |
| Extends without rewriting | **Strong** | Both follow-ups landed on real seams with no restructuring. Correct instinct that tier is per-`ShowSeat` and promotion is per-`MovieShow`. |

**Overall: strong mid-level, senior on modelling, mid-level on verification. Hire — not yet a senior strong-hire.** The object modelling and concurrency reasoning are already at the bar. The gap is not checking his own work before declaring it done — the identical gap his DSA rounds showed the same day. One habit, two formats.

### What a senior strong-hire would have done on THIS problem

**Proposed the operations and rules rather than asking for them.** Not *"what are the rules for reserving?"* but *"I'll assume a hold is time-boxed, multi-seat holds are all-or-nothing, and only the holder can convert a hold to a booking — correct me."* Same information, and it demonstrates domain knowledge. He had all three in his head; he asked instead of asserting.

**Established the concurrency posture in phase 1, not minute 46.** On a seat-booking problem, *"is this single-process, and are concurrent reservations in scope?"* is the second question out of your mouth, because the answer changes the class design — whether `ShowSeat` needs a lock, whether `holdSeat` is a compare-and-set, whether the aggregate can safely check-then-act at all. Asking at minute 46 means the design was already fixed by the time he learned the answer.

**Given `Hold` its own identity.** Hold state smeared across two `ShowSeat` fields works for the rules as stated, but `cancelBooking`, "release my hold early", and "show me my active holds" all need to name a hold. `Hold(id, user, showSeats, expiresAt)` makes each a one-liner instead of a scan. Not required by the given spec — it's the entity that makes the follow-ups free, and the follow-ups are always coming.

**Put the check in exactly one place:**
```
ShowSeat.canBeHeldBy(now) -> bool:
    if status == BOOKED: return false
    if status == RESERVED and now < holdExpiryTime: return false
    return true

MovieShow.holdSeats(user, seats, expiry):
    distinct = dedupe(seats)                       // the case that broke him
    validate every seat in distinct belongs to this.showSeats
    now = currentTime
    if any seat in distinct where not seat.canBeHeldBy(now):
        throw SeatNotHoldable(that seat)
    for seat in distinct:
        seat.holdSeat(user, expiry)                // now guaranteed to succeed
```
One rule, one owner; the aggregate asks a question rather than reading two fields; the dedupe makes the second loop's success a real invariant rather than an assumption. Note this is *not* Tell-Don't-Ask violated — `canBeHeldBy` returns a decision the seat made, not raw state the caller interprets. Exposing `status` is Ask; exposing `canBeHeldBy` is Tell.

**Traced it before being asked.** The trace takes forty seconds and the trigger is mechanical: *this method loops twice over the same collection and mutates in the second loop — what does the second loop see that the first didn't?* Every check-then-act over a collection has that question. He wrote exactly that structure twice and never asked it once.

**Added `BookingStatus` when cancellation came up.** `cancelBooking` as a method name isn't an answer. The answer is `Booking` gains `status: CONFIRMED | CANCELLED`, `MovieShow` owns the 30-minute check because `MovieShow` owns `time`, and cancellation calls `freeSeat()` on each seat.

### Drill

For the next three LLD rounds, before submitting any method that **loops twice over the same collection** — validate-all-then-apply-all, the standard atomic-operation shape, written in almost every resource-allocation problem — write these two lines above it and answer them:
```
// Loop 2 sees state loop 1 created. What state, and does loop 1's check still hold?
// What does this collection look like if it has duplicates?
```
Not a general "trace your code" resolution — a specific trigger attached to a specific code shape he writes constantly. Today it would have taken forty seconds and moved this round from a 2 to a 4.

Then the verification section of `lld_senior_guidance.md` — the checklist he keeps skipping, and now the only thing standing between his designs and a senior score.
