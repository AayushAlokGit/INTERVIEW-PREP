# LLD Sprint Transcript (front half, timeboxed)
**Date:** 2026-08-17
**Start Time:** 11:16:24 · **End Time:** 11:46:10
**Problem:** Movie ticket booking system — users browse shows and book seats for them
**Category:** resource allocation (concurrency-first)
**Difficulty:** 4/5 — the hold/expiry lifecycle plus atomic multi-seat booking under contention
**Front-half readiness: 2/5**
**Complete inside 22:00: no** — class design was still being composed at the buzzer; it landed at 29:46 and even then `User`, `Seat`, `Movie`, `ShowSeat`, `Booking` were entity lines, not classes
**Out of Scope list produced:** Unprompted (UI, durability, payment) — first time in 6 sessions
**Orchestrator named:** Yes (`BookingSystem`)
**Entity revision passes:** 0
**Untyped state or signatures:** `bookSeat(seatId)`, `reserveSeart(seatId)`, `reserveSeat(MovieShow, seatId)`, `bookSeat(MovieShow, seatId)` — no parameter types, no return types on any mutator; `getShowsByMovieOrCity(Movie, cityName)` half-typed

| Phase | Pace target | Landed at | ± vs target | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 6:00 | 6:39 | +0:39 | 4 (3 question, 1 submission) | 3/5 |
| Entities & relationships | 10:00 | 14:25 | +4:25 | 1 | 3/5 |
| Class design | 22:00 | 29:46 | +7:46 | 1 | 2/5 |

## Walk coverage
| # | Item | Hit/Partial/Miss | Evidence |
|---|---|---|---|
| 1 | Actors & entry point | Partial | "Actors -> User" only; no second actor probed; entry point unnamed until `BookingSystem` appeared in entities |
| 2 | Core operations | Hit | Asked responsibilities; recorded search by movie+city, view availability, book seats |
| 3 | Rules & legality | Hit | Asked search and booking rules; "a seat for a show can be booked by one user only" |
| 4 | Lifecycle & terminal states | Hit | Asked seat transitions unprompted; AVAILABLE → RESERVED → BOOKED with release on timeout |
| 5 | Failure behaviour | Partial | Never stated as a requirement; a consistent throw convention appears only implicitly in class-design signatures at 29:46 |
| 6 | Multiplicity & domain variants | Miss | Never asked — no screens, no multi-seat booking, no pricing |
| 7 | Concurrency posture | Hit (asked only) | "Is this single threaded or multi threaded?" → requirement 5 written; but no lock, granularity, or critical section anywhere in the design |
| 8 | Explicit Out of Scope | Hit | UI, durability, payment — unprompted |

**Budget allocation:** 0:00–6:39 requirements (3 serialized clarifying-question messages, 4:20 of the 6:39 spent on round-trips); 6:39–14:25 entities (7:46 for an 8-line list); 14:25–29:46 class design (15:21, still incomplete). 22:00 buzzer fell mid-composition of class design.

**First-pass completeness:** One submission per phase, zero back-fill, zero revisions. Nothing message two had to rescue. The front half is slow because of composition throughput, not incomplete first passes — the same finding as 2026-08-13. `ShowSeat (id, seat, status, reservationTime)` was typed once in entities and re-typed in class design; the same eight lines were paid for twice.

**Silent assumptions:** One theatre = one seat pool (no screens) — would have cost a full re-model in a real round. One seat per booking call — the atomic group-booking rule, the hardest requirement in this domain, never entered the design. Uniform pricing. Payment result reported back by someone unnamed (no method transitions RESERVED → BOOKED for a specific user).

**Dangling rules:** None left unresolved in requirements — reservation timeout, one-user-per-seat, and search rules all reached a resolution.

**Untraceable state / unenforceable requirements:**
- Requirement 5 (concurrent booking of the same seat) — enforced by nothing: no lock, no compare-and-set, no named critical section.
- Requirement 4 (hold expires) — `reservationTime:dateTime` is inert: no method reads it, no sweeper, no lazy expiry check in any signature.
- "Reserved *for a user*" — `ShowSeat` has no `reservedBy`, so any second caller can `bookSeat` a seat another user holds, re-introducing the exact violation requirement 3 forbids.
- `Booking.user:User` is unfillable — no method takes a user, and no signature returns a `Booking`, so nothing constructs one.
- Fields no requirement needs: `Theatre.seats[]` (redundant once `ShowSeat` references `Seat`), `Movie.time`; `Theatre.location` appeared in entities and silently vanished in class design.

## What he produced (verbatim, as it stood at 22:00)

### Requirements
```
1. Actors -> User
2. Operations -> users search shows by movie and city , view seat availability for show , book seats.
3. A seat for a show can be booked by one user only.
4. Seat reserved for some time for a user while they are making the payment. If payment not made within timeout seat released. When payment made then seat is booked.
5. Multi threaded should handle multiple people trying to book the same seat.
```

Clarifying questions asked (in order): what are the responsibilities expected of the system · rules for searching shows and booking seats · what state transitions occur for the seat · single or multi threaded · in memory or DB · do we need seat holds while the user is checking · do we need checkout.

### Out of Scope
```
1. UI
2. Durability
3. Payment
```

### Entities & relationships
```
1. User
2. Theatre (city: City, seats:Seat[], location, movieShows:MovieShows[])
3. Seat (id)
4. MovieShow(showTime, Movie, ShowSeats:ShowSeat[])
5. Movie (id, name, time)
6. ShowSeat (id, seat, status: AVAILABLE | RESERVED | BOOKED)
7. Booking(id, movieShow:MovieShow , showSeat:ShowSeat)
8. BookingSystem (theatres:[] )
```

### Class design
```
1. User

class Theatre :
-city:string
-seats:Seat[],
-movieShows:MovieShows[]
+getCity() -> string
+getShowForMovie(Movie) -> MovieShow | null

3. Seat (id)

class MovieShow :
-showTime: datetime
-movie:Movie
-showSeats:ShowSeat[]
-theatre: Theatre
+getShowTIme() -> datetime
+getAvailableSeats() -> ShowSeat[]
+bookSeat(seatId) -> throw exception if seat already booked
+reserveSeart(seatId) -> throw exception if seat already reserved.

5. Movie (id, name, time)
6. ShowSeat (id, seat, status: AVAILABLE | RESERVED | BOOKED, reservationTime:dateTime)
7. Booking(id, movieShow:MovieShow , showSeat:ShowSeat, user:User)

class BookingSystem :
-theatres:Theatres[]
-movieShowMap:Map<string,MovieShows[]>
-seatReservationTimeout: int
+getShowsByMovieOrCity(Movie, cityName) -> MovieShows[] ,throws exception if Movie not present in system
+getAvailableSeatsForShow(MovieShow) -> ShowSeats[] , throws exception if show is invalid (in past)
+reserveSeat(MovieShow, seatId) throws exception if seat does not exist of show is invalid
+bookSeat(MovieShow, seatId) , throws exception if seat does not exist of show is invalid
```

## What was still missing at 22:00
The whole class-design block — it arrived at 29:46. Beyond the overrun, still absent at submission: any `Screen` concept; group/multi-seat booking; a `Hold`/`Reservation` object with an owner and an expiry; a hold owner on `ShowSeat`; any locking or atomicity mechanism for requirement 5; any expiry mechanism for requirement 4; any method that constructs or returns a `Booking`; return types on every mutator; typed state and methods for `User`, `Seat`, `Movie`, `ShowSeat`, `Booking`; a stated split between the duplicated `reserveSeat`/`bookSeat` on `MovieShow` and `BookingSystem`.

## Where the time went
4:20 of the first 6:39 went to three serialized clarifying-question messages — the questions were good, the round-trips were the cost. Entities then took 7:46 for eight lines that needed no revision, and class design took 15:21 while re-typing entity fields already written once. Nothing was rebuilt or re-derived; the checklist was being *filled in*, not reconstructed. This is a throughput failure, not a thinking failure — and it is the second consecutive session where the last phase was cut by composition speed.

Notable positives worth preserving: item 4 (lifecycle) was asked unprompted and answered precisely, `ShowSeat` — the entity that exists only to hold the per-show rule — was reached for on the first pass, the entity list needed zero revisions, and the Out of Scope list appeared without any prompt for the first time in six sessions.

## Ideal front half (writable in the same 22 minutes)

### Requirements (target 5:00)
1. **Actors & entry point.** One actor: `User`. Single entry point `BookingService`. Catalogue pre-loaded, not created through the API.
2. **Core operations.** `searchShows(movieId, city)` · `getAvailableSeats(showId)` · `hold(showId, seatIds, userId)` · `confirm(holdId)`.
3. **Rules.** (a) A seat in a show is held or booked by at most one user. (b) A hold is all-or-nothing across the requested seats. (c) Only the holding user may confirm. (d) A show in the past accepts nothing.
4. **Lifecycle.** ShowSeat: `AVAILABLE → HELD → BOOKED` (terminal). `HELD → AVAILABLE` on TTL expiry, checked lazily on read and on hold attempt — no background thread.
5. **Failure convention (one, held everywhere).** Mutations throw `BookingException` subtypes (`SeatUnavailable`, `HoldExpired`, `ShowNotFound`); reads return empty collections, never null.
6. **Multiplicity & variants.** A theatre has many screens; a screen has fixed seats; a show is (movie × screen × time). Seats are booked in **groups** — this is what forces 3(b). Uniform pricing; tiers out of scope.
7. **Concurrency posture.** Multi-threaded, many users per show. One lock per `Show`, held across hold and confirm. Cost: serialises bookings for one popular show, but the show is the only correctness boundary and per-seat locks make all-or-nothing deadlock-prone.
8. **Out of Scope.** UI · persistence · payment gateway · cancellation & refunds · pricing tiers · auth.

> **What this buys:** item 6 produces `Screen` and, decisively, group booking — which turns 3(b) into the design's spine. Naming the failure convention in item 5 means every later signature is already decided instead of improvised at minute 25.

### Entities & relationships (target 9:00)
```
Theatre ──owns──> Screen ──has──> Seat            (physical, immutable)
Movie
Show ──owns──> ShowSeat ──refs──> Seat            (Show = Movie × Screen × time)
Hold ──refs──> Show, seatIds, userId, expiresAt
Booking ──refs──> Hold, userId, seatIds
BookingService  ← orchestrator: catalogue lookup, lock acquisition, hold/booking registries
```
**Entities that exist only to hold a rule:** `ShowSeat`, forced by 3(a) — availability is per-show, not per-seat. And `Hold`, forced by 3(c) and lifecycle 4 — expiry needs one `expiresAt` for the whole group and confirm needs an owner. A `reservationTime` field on `ShowSeat` can express neither: no owner, no group.

> **What this buys:** `Screen` fixes the multiplex; `Hold` is precisely the object his version lacked, and its absence is what left requirements 4 and 3(c) with nothing to enforce them.

### Class design (target 22:00)
```
class Show                                    // rules 3(a),(b),(d) — it owns every ShowSeat
  - id: String                                // R2
  - movieId: String                           // R2
  - screenId: String                          // R6
  - startsAt: Instant                         // R3(d)
  - seats: Map<String, ShowSeat>              // R3(a)
  - lock: ReentrantLock                       // R7
  + isBookable(now: Instant): boolean                              // R3(d)
  + availableSeats(now: Instant): List<String>                     // R2, R5 empty-not-null
  + holdSeats(seatIds: List<String>, userId: String, ttl: Duration,
              now: Instant): Hold             // throws SeatUnavailable — R3(a),(b) atomic under lock
  + confirmHold(hold: Hold, now: Instant): void  // throws HoldExpired — R3(c), R4

class ShowSeat                                // owns its own transition (Tell, don't Ask)
  - seatId: String                            // R6
  - status: SeatStatus                        // R4
  - holdId: String | null                     // R3(c)
  + isAvailable(now: Instant): boolean                             // R4 lazy expiry
  + markHeld(holdId: String): void            // throws SeatUnavailable — R3(a)
  + markBooked(holdId: String): void          // throws SeatUnavailable — R3(c)
  + release(): void                                                // R4

class Hold
  - id: String                                // R3(c)
  - showId: String                            // R2
  - userId: String                            // R3(c)
  - seatIds: List<String>                     // R3(b)
  - expiresAt: Instant                        // R4
  + isExpired(now: Instant): boolean                               // R4

class Booking
  - id: String  - showId: String  - userId: String
  - seatIds: List<String>  - bookedAt: Instant                     // R2

class Screen
  - id: String  - theatreId: String  - seatIds: List<String>       // R6

class BookingService                          // orchestrator: workflow only, no seat rules
  - showsById: Map<String, Show>                                   // R2
  - showsByMovieAndCity: Map<String, List<String>>                 // R2 search index
  - holdsById: Map<String, Hold>                                   // R3(c)
  - bookingsById: Map<String, Booking>                             // R2
  - holdTtl: Duration                                              // R4
  + searchShows(movieId: String, city: String): List<Show>         // R2, empty if none
  + availableSeats(showId: String): List<String>                   // throws ShowNotFound
  + hold(showId: String, seatIds: List<String>, userId: String): Hold
  + confirm(holdId: String, userId: String): Booking               // throws HoldExpired
```
**Why each rule sits where it sits:** all-or-nothing (3b) and the lock live on `Show` because only `Show` sees every seat in the group — no smaller scope can make the group atomic. Per-seat legality (3a) and expiry (4) live on `ShowSeat` because it owns `status`; `Show` calls `markHeld` and never reads-then-writes status. Ownership (3c) is checked inside `Show.confirmHold` via `Hold`, not in the service, because it is a domain rule rather than workflow. `BookingService` holds only indexes and registries; every method is lookup plus delegate.

> **What this buys:** each field tagged with its requirement number makes the two dead fields in his version impossible to keep, and requirement 7 finally has a named home (`Show.lock`) instead of being a sentence with no code behind it.

## Feedback given
- The box was overrun by 35%: class design landed at 29:46. In a real 40-minute round there would have been negative eight minutes for implementation.
- Item 8 (Out of Scope) appeared unprompted for the first time in six sessions — a genuine break in the pattern. Item 4 (lifecycle) was the strongest work of the round, asked without a nudge.
- Item 6 was never asked, and that single miss cost three defects: no `Screen`, no multi-seat atomic booking, no pricing.
- Requirements 4, 5 and "reserved for a user" were all stated and then enforced by nothing. Asking the concurrency question and not acting on the answer is worse than not asking — it is on the record.
- No mutator declared a return type; nothing constructs the `Booking` the design contains.
- The one habit to change: **ask item 6 and item 7 back to back in the very first question message, before anything else.**

**Sitting note:** problem 2 (shared expense splitter, 3/5) was served at 11:46:10 and paused by the candidate at 0:40 with no content submitted. Not scored.
