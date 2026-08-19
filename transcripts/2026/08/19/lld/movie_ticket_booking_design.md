# Movie Ticket Booking with Seat Holds — Design Sheet

## 1. Requirements

**Actors:** Users, BookingSystem

**Operations:**
- Users browse shows for a movie
- Users check available seats
- Users can reserve seats for some time period before checkout
- Users can book the tickets for a movie show

**Rules:**
- Seat can be held for 5 minutes
- Seats become available when holds expire
- User can only book the seats they currently hold, and only their held seats
- Holds are acquired atomically over all requested seats

**Seat states (per show):** `AVAILABLE`, `RESERVED`, `BOOKED`

**Structure:** Multiple theatres, each having multiple screens (with fixed layout); different movies play on the screen at different times.

## 2. Out of Scope
1. UI
2. Movie show addition

## 3. Entities & Relationships

```
Movie(name)
Theatre(List<Screen>)
Screen(List<Seat>)
Seat(id, type)
MovieShow(time, movie: Movie, screen: Screen, theatre: Theatre, List<ShowSeat>)
ShowSeat(seat: Seat, status: SeatStatus, holdAcquiredBy: User|null, holdExpiryTime: time|null)
Enum SeatStatus: AVAILABLE | RESERVED | BOOKED
Booking(movieShow: MovieShow, seats: List<ShowSeat>, amount, createdAt, user: User)
BookingSystem(Map<string, List<MovieShow>>)   // as written; keying not yet stated
```

## 4. Class Design

> Transcribed exactly as described, including signatures as given. `Booking.amount` was dropped
> during the entity probe ("no requirement needs it — payment is out of scope").

```
class BookingSystem:
  - holdExpiryTimeout: int
  - movieShowMap: Map<string, List<MovieShow>>        // key = movie name
  + getShowsForMovie(movieName: string) -> List<MovieShow>
        // throws MovieNameNotFound
  + getAvailableSeats(movieShow: MovieShow) -> List<ShowSeat>
        // throws InvalidShow if time in past
  + reserveSeats(movieShow: MovieShow, showSeats: List<ShowSeat>, acquiringUser: User) -> void
        // throws exception if seat booked or already reserved
  + bookSeats(movieShow: MovieShow, showSeats: List<ShowSeat>) -> Booking
        // throws SeatHoldNotAcquired, SeatHoldExpired

class ShowSeat:
  - id: string
  - type: string
  - status: SeatStatus
  - holdAcquiredBy: User | null
  - holdExpiryTime: time | null
  + holdSeat(user: User, holdExpiryTime: time) -> void
  + freeSeat() -> void
  + bookSeat(user) -> void         // throws exception if user != holdAcquiredBy
  + isAvailable() -> bool

class MovieShow:
  - showSeats: List<ShowSeat>
  - time
  - theatre: Theatre
  - screen: Screen
  + getAvailableSeats() -> List<ShowSeat>
  + holdSeat(user: User, seats: List<ShowSeat>, holdExpiryTime) -> void
  + bookSeat(user: User, seats: List<ShowSeat>) -> Booking

enum SeatStatus: AVAILABLE | RESERVED | BOOKED
```

### Amendment made during probing
`BookingSystem.bookSeats` gains a user parameter:
```
  + bookSeats(movieShow: MovieShow, showSeats: List<ShowSeat>, user: User) -> Booking
```

**Expiry policy (stated verbally):** lazy — "the status of a held seat changes only on reads, so the
seats will be in RESERVED status and will remain in RESERVED state until read at a point after 10:05."

## 5. Core Implementation (pseudo-code)

### BookingSystem — described in prose
```
BookingSystem.reserveSeats:
    Check the current status of the given show seats; if any of them not available then throw exception.
    holdExpiryTime = now + holdExpiryTimeout
    Call movieShow.holdSeats(user, List<ShowSeat>, holdExpiryTime) to change status of all seats to reserved.

BookingSystem.bookSeats:
    Check that the user initiating the booking request has acquired an unexpired hold on the seats.
    If not then throw exception.
    Call movieShow.bookSeats to change seat status and generate booking object.
```

### ShowSeat
```
holdSeat(user, expiryTime):

    if status == BOOKED:
        throw SeatAlreadyBooked

    if status == RESERVED:
        if currentTime < holdExpiryTime:
            throw SeatAlreadyReserved

        // Hold expired
        freeSeat()

    status = RESERVED
    holdAcquiredBy = user
    holdExpiryTime = expiryTime


freeSeat():

    status = AVAILABLE
    holdAcquiredBy = null
    holdExpiryTime = null


bookSeat(user):

    if status != RESERVED:
        throw SeatHoldNotAcquired

    if currentTime >= holdExpiryTime:
        freeSeat()
        throw SeatHoldExpired

    if holdAcquiredBy != user:
        throw SeatNotHeldByUser

    status = BOOKED
    holdAcquiredBy = null
    holdExpiryTime = null
```

### MovieShow
```
holdSeat(user, seats, expiryTime):

    // First check ALL seats
    for seat in seats:

        if seat.status == BOOKED:
            throw SeatAlreadyBooked

        if seat.status == RESERVED
           and currentTime < seat.holdExpiryTime:
            throw SeatAlreadyReserved

    // All seats are available
    // Now hold ALL of them
    for seat in seats:
        seat.holdSeat(user, expiryTime)


bookSeat(user, seats):

    // Validate ALL seats first
    for seat in seats:

        if seat.status != RESERVED:
            throw SeatHoldNotAcquired

        if currentTime >= seat.holdExpiryTime:
            throw SeatHoldExpired

        if seat.holdAcquiredBy != user:
            throw SeatNotHeldByUser

    // Book ALL seats
    for seat in seats:
        seat.bookSeat(user)

    return new Booking(
        movieShow = this,
        seats = seats,
        user = user,
        createdAt = currentTime
    )
```

## 6. Extensibility follow-ups (as answered)

**Seat tiers + per-tier promotions:** "ShowSeat will have the tier information for the seat for that
particular show. And promotions are also a show level detail so should be present in MovieShow."

**Booking cancellation (up to 30 min before showtime):** "another method `cancelBooking` to be added
to BookingSystem, MovieShow."

---
---

# Optimal Reference (what a senior strong-hire would design)

> Everything above section 6 is exactly what was described in the round, gaps included.
> Everything below is the teaching the round withheld.

## 1. Requirements + Out of Scope

**Actors:** Customer (browses, holds, books, cancels) · Admin (creates shows — out of scope) ·
**Orchestrator:** `BookingService`

**Core operations**
- `searchShows(movieName, [theatre], [date])` → shows
- `getAvailableSeats(showId)` → seats currently holdable
- `holdSeats(showId, seatIds, user)` → `Hold` (atomic across all seats)
- `releaseHold(holdId, user)` — user abandons checkout
- `confirmBooking(holdId, user)` → `Booking`
- `cancelBooking(bookingId, user)`

**Rules & legality**
- A hold is time-boxed (5 min) and expires with no user action.
- Multi-seat holds are **all-or-nothing**.
- Only the holder may convert a hold to a booking; only the booker may cancel.
- A seat cannot be held or booked once the show has started.
- Cancellation permitted up to 30 min before showtime.
- Max N seats per hold (a question never asked; without it one user can hold a whole screen).

**Lifecycle**
- `ShowSeat`: `AVAILABLE → HELD → BOOKED`; `HELD → AVAILABLE` on expiry or release;
  `BOOKED → AVAILABLE` on cancellation. **`BOOKED` is not terminal** — cancellation makes it
  reversible, which is precisely why `Booking` needs its own state.
- `Hold`: `ACTIVE → CONSUMED | RELEASED | EXPIRED`
- `Booking`: `CONFIRMED → CANCELLED`

**Failure convention:** exceptions, one per distinct cause, never collapsed —
`ShowNotFound`, `SeatNotInShow`, `SeatNotHoldable(seatId)`, `HoldNotFound`, `HoldExpired`,
`HoldNotOwnedByUser`, `ShowAlreadyStarted`, `CancellationWindowClosed`, `TooManySeatsRequested`.
*(He got this right in substance — seven specific exceptions held consistently across every
signature. Keep doing it, but state it in phase 1 so it reads as a decision rather than an
accident.)*

**Multiplicity & variants:** many theatres → many screens → fixed seat layout; seat tiers with
per-show pricing; per-tier promotions; multiple concurrent holds per user across different shows.

**Concurrency posture (the item missed in phase 1):** single process, **multi-threaded**. Concurrent
holds on the same seat are the central risk and must be in scope from the start, because the answer
changes the class design — see §7.

**Out of Scope:** UI · show/theatre administration · payment processing and refunds · user auth ·
persistence · pricing engine beyond tier + promotion · notifications · waitlists.

## 2. Entities & relationships

```
Movie 1 ────< Show >──── 1 Screen ────< Seat        (Seat = physical, immutable, belongs to Screen)
                |
                +──< ShowSeat  ──1── Seat           (ShowSeat = per-show mutable state)
                              ──0..1─ Hold
User  1 ──< Hold    >── * ShowSeat
User  1 ──< Booking >── * ShowSeat
BookingService ── owns ──> shows, holds, bookings
```

**Orchestrator:** `BookingService` — the only public entry point. It resolves ids to objects and
delegates; it holds **no rule**.

**The modelling decision that matters — and he got it unaided:** `Seat` is physical and immutable;
`ShowSeat` carries the per-show mutable state. Conflating them is the classic failure here.

**The entity he missed:** `Hold` as a first-class object. Not strictly required by the stated rules —
his two `ShowSeat` fields cover them — but it is what makes `releaseHold`, `cancelBooking`, "my
active holds", and per-hold expiry into one-liners instead of scans over every seat in every show.

## 3. Class outlines

```
class Seat:                                   // physical, immutable
  - id: SeatId
  - row: int
  - number: int
  - tier: SeatTier
  + label(): string

enum SeatTier:      STANDARD | PREMIUM | RECLINER
enum SeatStatus:    AVAILABLE | HELD | BOOKED
enum HoldStatus:    ACTIVE | CONSUMED | RELEASED | EXPIRED
enum BookingStatus: CONFIRMED | CANCELLED

class ShowSeat:                               // per-show mutable state
  - seat: Seat
  - status: SeatStatus
  - holdId: HoldId | null
  - holdExpiresAt: Instant | null
  - price: Money                              // tier price fixed at show creation
  + canBeHeldAt(now: Instant): bool           // a DECISION, not raw state   <-- the Tell fix
  + hold(holdId: HoldId, expiresAt: Instant): void
  + release(): void
  + book(): void
  + tier(): SeatTier
  + priceWith(promo: Promotion | null): Money

class Hold:
  - id: HoldId
  - user: UserId
  - showId: ShowId
  - seatIds: Set<SeatId>
  - expiresAt: Instant
  - status: HoldStatus
  + isActiveAt(now: Instant): bool
  + assertOwnedBy(user: UserId): void         // throws HoldNotOwnedByUser
  + consume(): void
  + release(): void

class Show:                                   // aggregate root for seat state
  - id: ShowId
  - movie: Movie
  - screen: Screen
  - startsAt: Instant
  - showSeats: Map<SeatId, ShowSeat>
  - promotion: Promotion | null
  - seatLocks: Map<SeatId, Lock>              // see §7
  + availableSeats(now: Instant): List<ShowSeat>
  + holdSeats(seatIds: Set<SeatId>, holdId: HoldId, expiresAt: Instant, now: Instant): void
  + releaseSeats(seatIds: Set<SeatId>): void
  + bookSeats(seatIds: Set<SeatId>): Money
  + hasStarted(now: Instant): bool
  + isCancellableAt(now: Instant): bool       // owns the 30-min rule; it owns startsAt

class Promotion:
  - tier: SeatTier
  - percentOff: int
  + applyTo(price: Money, tier: SeatTier): Money

class Booking:
  - id: BookingId
  - showId: ShowId
  - user: UserId
  - seatIds: Set<SeatId>
  - total: Money
  - status: BookingStatus                     // the field cancellation needs
  - createdAt: Instant
  + assertOwnedBy(user: UserId): void
  + cancel(): void                            // throws IllegalState if already CANCELLED

class BookingService:                         // orchestrator — no rules
  - shows: Map<ShowId, Show>
  - showsByMovie: Map<string, List<ShowId>>
  - holds: Map<HoldId, Hold>
  - bookings: Map<BookingId, Booking>
  - holdDuration: Duration
  - maxSeatsPerHold: int
  - clock: Clock                              // injected — makes expiry testable
  + searchShows(movieName: string): List<Show>
  + getAvailableSeats(showId: ShowId): List<ShowSeat>
  + holdSeats(showId: ShowId, seatIds: Set<SeatId>, user: UserId): Hold
  + releaseHold(holdId: HoldId, user: UserId): void
  + confirmBooking(holdId: HoldId, user: UserId): Booking
  + cancelBooking(bookingId: BookingId, user: UserId): void
```

**Note the parameter types.** Public methods take **ids**, not live `ShowSeat` objects, and seat
collections are **`Set`**, not `List`. That closes three holes at once: seats from another show
cannot be passed in, the caller cannot hold a stale reference across a hold expiry, and the
duplicate-seat bug becomes *unrepresentable* rather than something to remember to handle.

## 4. Core method implementations (pseudo-code)

```
BookingService.holdSeats(showId, seatIds, user):
    show = shows[showId] ?? throw ShowNotFound
    now  = clock.now()

    if show.hasStarted(now):             throw ShowAlreadyStarted
    if seatIds.isEmpty():                throw NoSeatsRequested
    if seatIds.size > maxSeatsPerHold:   throw TooManySeatsRequested

    holdId    = HoldId.generate()
    expiresAt = now + holdDuration
    show.holdSeats(seatIds, holdId, expiresAt, now)    // atomic; throws if any seat unholdable

    hold = new Hold(holdId, user, showId, seatIds, expiresAt, ACTIVE)
    holds[holdId] = hold
    return hold


Show.holdSeats(seatIds, holdId, expiresAt, now):
    // seatIds is a SET - duplicates cannot exist. This is the fix, and it is structural.
    seats = []
    for id in seatIds:
        s = showSeats[id] ?? throw SeatNotInShow(id)   // validates membership
        seats.add(s)

    withSeatLocksInIdOrder(seatIds):                   // §7 - consistent order, no deadlock
        // PHASE 1 - decide. Ask each seat for its verdict; never read its fields.
        for s in seats:
            if not s.canBeHeldAt(now): throw SeatNotHoldable(s.seat.id)

        // PHASE 2 - apply. Guaranteed to succeed: set input, locks held, verdicts fresh.
        for s in seats:
            s.hold(holdId, expiresAt)


ShowSeat.canBeHeldAt(now):
    if status == BOOKED: return false
    if status == HELD:
        if now < holdExpiresAt: return false
        release()                                      // lazy expiry, reclaimed here
    return true


ShowSeat.hold(holdId, expiresAt):
    if status != AVAILABLE: throw IllegalState         // invariant guard, NOT a second copy
    status        = HELD                               // of the holdability rule
    this.holdId   = holdId
    holdExpiresAt = expiresAt


BookingService.confirmBooking(holdId, user):
    hold = holds[holdId] ?? throw HoldNotFound
    hold.assertOwnedBy(user)                           // Tell - the hold checks its own ownership

    now  = clock.now()
    show = shows[hold.showId]

    if not hold.isActiveAt(now):
        show.releaseSeats(hold.seatIds)
        hold.release()
        throw HoldExpired

    total = show.bookSeats(hold.seatIds)               // ShowSeat.book(): HELD -> BOOKED
    hold.consume()

    booking = new Booking(BookingId.generate(), hold.showId, user, hold.seatIds,
                          total, CONFIRMED, now)
    bookings[booking.id] = booking
    return booking


BookingService.cancelBooking(bookingId, user):
    booking = bookings[bookingId] ?? throw BookingNotFound
    booking.assertOwnedBy(user)

    show = shows[booking.showId]
    if not show.isCancellableAt(clock.now()): throw CancellationWindowClosed

    booking.cancel()                                   // CONFIRMED -> CANCELLED
    show.releaseSeats(booking.seatIds)                 // BOOKED -> AVAILABLE
```

**Edge cases covered that the round's version did not:**
duplicate seat ids (impossible — `Set`) · seats not belonging to the show (validated) ·
empty seat list · seat-count cap · show already started · a hold expiring between
`getAvailableSeats` and `holdSeats` (revalidated inside the lock) · double-cancel ·
cancelling after the window · confirming an already-consumed hold.

## 5. Design decisions (each vs its named alternative)

| Decision | Alternative | What it gives up |
|---|---|---|
| `ShowSeat` separate from `Seat` | one `Seat` carrying status | Nothing. The alternative is simply wrong once a screen hosts two shows. **He got this right, unaided.** |
| `Hold` as an entity | hold fields on `ShowSeat` (**his choice**) | One more class and a registry. Buys: `releaseHold`, "my holds", per-hold expiry, and an id to pass to `confirmBooking` instead of re-passing the seat list. His version works for the stated rules; it stops working at the first follow-up. |
| Public API takes **ids** and **`Set`** | takes `List<ShowSeat>` objects (**his choice**) | A map lookup per call. Buys: no foreign seats, no stale references, and the duplicate bug becomes unrepresentable. |
| `canBeHeldAt()` returns a decision | expose `status` + `holdExpiryTime` (**his choice**) | Nothing. Same information, but the rule lives in one place — exactly what would have prevented the bug. |
| Lazy expiry | background sweeper thread | Held seats look unavailable until touched; memory isn't reclaimed. Buys: no thread, no clock skew, no sweep interval to tune. **His choice, and the right one at this scale** — a sweeper only earns its keep when stale holds must be *visible* as free without a read. |
| Injected `Clock` | `currentTime` inline (**his choice**) | Nothing. Buys: expiry becomes testable without sleeping. |
| **No pattern** for tiers/promotions | Strategy per pricing rule | With one promotion shape, a `Promotion` value object with `applyTo` is enough. Reach for Strategy at the third pricing rule, not the first. **He forced no patterns all round — that judgement is already correct.** |

## 6. Extensibility — where each follow-up lands

- **Seat tiers + per-tier promotions.** `tier` on `Seat` (physical — a recliner is a recliner in every
  show), `price` on `ShowSeat` (per-show), `promotion` on `Show`. `Show.bookSeats` sums
  `showSeat.priceWith(promotion)`. **His answer landed on the right seams**; only the split between
  physical tier and per-show price was missing.
- **Cancellation.** `BookingStatus` on `Booking`, the 30-min rule on `Show` (it owns `startsAt`),
  and `Show.releaseSeats` already exists from the hold path. No new mechanism — which is the test of
  whether the seams were right.
- **Waitlist.** A `Waitlist` subscribing to seat-release events from `Show`. `releaseSeats` is the
  single publication point — one call site, because release was already centralised.
- **Dynamic pricing.** `ShowSeat.priceWith` is already the single pricing seam; swap `Promotion` for
  a `PricingPolicy` interface only when a second rule shape actually appears.

## 7. Concurrency

**Category: correctness — a lost update on a check-then-act.** Two threads both read
`status == AVAILABLE`, both write `HELD`, and the second silently overwrites the first's hold.
Naming the *category* matters because it selects the primitive: a correctness failure needs mutual
exclusion over the read-modify-write; a *scarcity* failure would need a semaphore; a *coordination*
failure a condition variable. Reaching for a lock without naming which one you have is how people
end up with a lock that doesn't fix anything.

**Smallest sufficient primitive:** a per-`ShowSeat` lock, held across the decide-and-apply span.
Not a global lock (serialises the whole system), not a per-`Show` lock (serialises a 300-seat screen
on a release day), not per-seat atomics (they cannot make a *multi-seat* hold atomic).

**Where the synchronization lives:** inside `Show.holdSeats` / `bookSeats` / `releaseSeats` — the
aggregate that owns the seat map. Never in `BookingService`, and never exposed to callers. The
critical section is exactly the decide-then-apply span and nothing else.

**The multi-seat consequence — the follow-up he answered correctly:** an atomic hold needs *all*
seat locks held simultaneously, so acquisition order matters. **Always acquire in a globally
consistent order — sorted by seat id** — or two users each holding one of a pair deadlock.

```
Show.withSeatLocksInIdOrder(seatIds, body):
    ordered = seatIds.sorted()                     // the deadlock fix
    for id in ordered: seatLocks[id].acquire()
    try:     body()
    finally: for id in ordered.reversed(): seatLocks[id].release()
```

**Cost, stated:** N lock acquisitions per hold instead of one, plus a `Lock` object per seat in
memory. Throughput stays high because disjoint seat sets never contend — which was his own
justification, and it is the correct one.

**What still isn't safe, and shouldn't be:** `getAvailableSeats` is a snapshot the moment it
returns. That's intentional — the authoritative check happens inside the lock in `holdSeats`, and a
UI showing a seat that's gone by the time you click it is a product reality, not a bug. Worth saying
out loud, because an interviewer will ask whether you noticed.
