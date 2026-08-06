# Amazon Locker — LLD Design Sheet

## Phase 1 — Requirements

1. Delivery driver inputs packet details and system assigns a locker if any and also one time code. If no locker assigned packet not deposited.
2. Packet sizes — S, M, L and Locker sizes — S, M, L.
3. Packet fits any locker having locker size >= packet size.
4. Locker fits only 1 packet.
5. When packet placed in locker, customer notified with locker details and one time code to open locker.
6. If packet not taken by customer in 3 days, it is reclaimed and locker is emptied.
7. Customer extracts package using one time code which uniquely identifies a lcoker in the system.

## Out of Scope
_(none stated)_

## Phase 2 — Entities & Relationships

- **LockerManager** — `lockers: Locker[]`. Orchestrator; notifies the customer.
- **Locker** — `size: Size`, `packet: Packet`
  - `occupied: bool` dropped — presence of `packet` determines occupancy.
- **Packet** — `size: Size`, `customer: Customer`, `oneTimeCode: int`, `installTime: datetime` (time the packet was placed in the locker)
  - `oneTimeCode` and `installTime` moved here from `Locker`.
- **Customer** — `name`, `email`, `phone`, ...

**Enums**
- `Size: S | M | L`

## Phase 3 — Class Design

### LockerManager
```
- sizeToLockerMap: Map<Size, List<Locker>>   // holds ALL lockers of that size, not only free ones
- oneTimeCodeToLockerMap: Map<int, Locker>
- generateOneTimeCode(packet, locker): int
- notifyCustomer(packet)
- reclaimExpiredPackets()          // runs periodically; reclaims lockers whose
                                   // packets have not been claimed for 3 days
+ getLockerForPacket(packet): Locker | null
+ putPacketInLocker(): void
+ takeoutPacketForCode(oneTimeCode): Packet | null
```
Caller flow (driver): call `getLockerForPacket`; if null, skip; else call `putPacketInLocker`.

### Locker
```
- packet
- size
+ putPacket()
+ removePacket()
```

### Packet
```
- customer
- oneTimeCode: int | null
- size            // stated in entities, omitted from class list by oversight
- installTime     // stated in entities, omitted from class list by oversight
+ setOneTimeCode(oneTimeCode)
+ unsetOneTimeCode()
+ getCustomer(): Customer      // called by LockerManager to notify the customer
```

### Customer
_(no members stated)_

## Phase 4 — Core Implementation (pseudo-code)

```
getLockerForPacket(packet) ->
    start iterating through lockers of size = packet size
        if empty locker found, return locker
    else move up a size (while size <= L) and repeat
    return null if no empty locker found


putPacketInLocker(packet, locker) ->
    1. generate a one time code for packet + locker combo
    2. save oneTimeCode -> locker in map of LockerManager
    3. update install time and one time code in packet, and put in locker
    4. notify customer


takeoutPacketForCode(oneTimeCode) ->
    check if oneTimeCode exists in map; if it does not exist, throw an exception
        for wrong code
    check locker; if the locker does not have a packet, throw an exception
    remove packet from locker
    unset oneTimeCode and installTime in packet
    remove oneTimeCode -> locker mapping from the map
```

**Revisions after trace (prompted):**
- `takeoutPacketForCode` throws on error rather than returning null; signature is not `Packet | null`.
- Add a check in `takeoutPacketForCode` for `installTime` being older than 3 days, so an
  expired code is rejected even when the periodic reclaim job has not yet run.

## Phase 5 — Follow-ups

**1. Refrigerated lockers (perishable packages).**
Keep two maps — one of refrigerated lockers by size, one of normal lockers by size.
Perishable packets always pick from the refrigerated map. Non-perishable packets check
the normal map first, and fall back to the refrigerated map if nothing is found.

**2. Variable hold windows (perishable 1 day, standard 3, Prime 5).**
The number still lives in `Packet`, but store `expiryTime` instead of `installTime`.
`LockerManager` holds the rule and computes `expiryTime`.

**3. Concurrency — two drivers, one free Medium locker.**
The shared resource (the locker map) needs to be protected to avoid race conditions.
Add a lock before accessing the map; this serialises the threads. The lock must enclose
both operations — `getLockerForPacket` and `putPacketInLocker` — so that only one thread
is handed the locker and only one packet is assigned to it.

---
---

# Optimal Reference (what a senior strong-hire would design)

> This section is the teaching the round deliberately withheld. Everything above is
> exactly what Aayush described. Everything below includes what he missed.

## 1. Requirements + Out of Scope

**Functional**
1. A driver deposits a package: the system allocates a locker, generates a one-time
   code, records the deposit, and notifies the recipient. If nothing suitable is free,
   the deposit is rejected and nothing is mutated.
2. Packages and lockers each have a size: S < M < L. A package fits any locker of size
   >= its own.
3. Allocation picks the **smallest** fitting free locker, so large lockers are not
   wasted on small packages.
4. A locker holds at most one package.
5. A customer retrieves by entering the one-time code alone; codes are unique across
   the cabinet, so the code identifies the locker.
6. A package expires 3 days after deposit. An expired code is rejected at the kiosk
   **and** a periodic sweep reclaims the locker so it becomes free again.
7. Invalid code, expired code, and empty locker are all rejected. No lockout, no retry
   limit.

**Questions he never asked, and what they'd have changed**
- *"Is this single-process/single-threaded?"* — Either it lands in Out of Scope (and
  the concurrency curveball is answered in advance), or it forces atomic allocation
  from the start. This is the highest-value unasked question in the round.
- *"Can a driver be handed a locker and then walk away without using it?"* — The
  answer collapses the two-call API (`getLockerForPacket` + `putPacketInLocker`) into
  a single `depositPacket(packet): Receipt`. That one change removes both the
  untrusted-locker bug and the entire race condition.
- *"Is a code single-use? Can one customer have several packages at once?"* — Confirms
  code -> locker is a 1:1 mapping cleared on pickup, and that nothing is keyed by
  customer.
- *"What is the error contract — exceptions or null returns?"* — Would have prevented
  the `Packet | null` signature that contradicted a throwing body.
- *"How are codes generated, and what happens on collision?"* — Uniqueness is a stated
  requirement; something has to enforce it.

**Out of Scope** *(he produced no list at all — this is the ceiling that bound his score)*
- Persistence / database; the cabinet is in-memory for this design.
- Multiple locker locations; one cabinet, one `LockerManager`.
- Delivery channel for notifications (email/SMS) — behind a `Notifier` interface.
- Authentication of the driver, physical hardware/door control, payments.
- Package routing, returns, and refunds.
- Analytics, capacity planning, locker maintenance state.

## 2. Entities & relationships

```
LockerManager  (ORCHESTRATOR — owns allocation, code registry, expiry sweep)
   |
   |-- owns --> Locker[]        (knows its own size + occupancy; enforces "fits & free")
   |-- owns --> Map<code, Locker>
   |-- uses --> CodeGenerator   (unique code issuance)
   |-- uses --> Notifier        (interface; email/SMS behind it)
   |-- uses --> Clock           (injected; makes expiry testable)

Locker --holds--> Packet (0..1)
Packet --for----> Customer
Packet  knows its own expiryTime and answers isExpired(now)
```

`LockerManager` is the only public entry point (a Facade in effect). `Locker` and
`Packet` each defend one invariant. `Clock` and `Notifier` are injected so the whole
thing is testable without waiting three days or sending mail.

## 3. Class outlines

```
LockerManager
  - lockers: List<Locker>
  - codeToLocker: Map<string, Locker>
  - codeGenerator: CodeGenerator
  - notifier: Notifier
  - clock: Clock
  - lock: Mutex
  + depositPacket(packet: Packet): Receipt        // throws NoLockerAvailableError
  + retrievePacket(code: string): Packet          // throws InvalidCode/ExpiredCode
  + reclaimExpired(): List<Packet>                // called by the sweeper
  - findSmallestFittingFreeLocker(packet): Locker | null

Receipt
  - lockerId: string
  - code: string
  - expiryTime: DateTime

Locker
  - id: string
  - size: Size
  - packet: Packet | null
  + isFree(): bool
  + canHold(packet: Packet): bool         // size AND any future attribute
  + putPacket(packet: Packet): void       // throws if occupied or does not fit
  + removePacket(): Packet                // throws if empty

Packet
  - id: string
  - size: Size
  - perishable: bool
  - customer: Customer
  - depositedAt: DateTime | null
  - expiryTime: DateTime | null
  + markDeposited(now: DateTime): void    // computes its OWN expiryTime
  + isExpired(now: DateTime): bool
  + clearDeposit(): void
  + notifyRecipient(receipt: Receipt, notifier: Notifier): void   // Tell, don't Ask
  + holdDuration(): Duration              // perishable 1d, Prime 5d, else 3d

Customer
  - id: string
  - name: string
  - email: string
  - tier: Tier
  + notify(message: string, notifier: Notifier): void

CodeGenerator
  + generate(existing: Set<string>): string

Notifier   (interface)
  + send(customer: Customer, message: string): void

Clock      (interface)
  + now(): DateTime

Size: S | M | L        (ordered)
Tier: STANDARD | PRIME
```

Note what is **not** here: no Factory, no Singleton, no Builder, no Strategy class.
The only abstractions are `Notifier` and `Clock`, and both exist to make the thing
testable — not to look sophisticated.

## 4. Core method implementations (pseudo-code)

```
# --- Locker: owns "fits and free" -------------------------------------------

Locker.isFree():
    return this.packet == null

Locker.canHold(packet):
    return this.isFree() and packet.size <= this.size

Locker.putPacket(packet):
    if not this.isFree():
        throw LockerOccupiedError(this.id)
    if packet.size > this.size:
        throw PacketDoesNotFitError(packet.id, this.id)
    this.packet = packet

Locker.removePacket():
    if this.isFree():
        throw LockerEmptyError(this.id)
    p = this.packet
    this.packet = null
    return p


# --- Packet: owns its own expiry policy --------------------------------------

Packet.holdDuration():
    if this.perishable:                 return Duration(days=1)
    if this.customer.tier == PRIME:     return Duration(days=5)
    return Duration(days=3)

Packet.markDeposited(now):
    this.depositedAt = now
    this.expiryTime  = now + this.holdDuration()

Packet.isExpired(now):
    return this.expiryTime != null and now >= this.expiryTime


# --- LockerManager: allocation, codes, lifecycle ------------------------------

LockerManager.findSmallestFittingFreeLocker(packet):
    best = null
    for locker in this.lockers:
        if not locker.canHold(packet):        continue
        if best == null or locker.size < best.size:
            best = locker
    return best

LockerManager.depositPacket(packet):
    with this.lock:                                   # check-then-act, atomic
        locker = this.findSmallestFittingFreeLocker(packet)
        if locker == null:
            throw NoLockerAvailableError(packet.size)

        code = this.codeGenerator.generate(this.codeToLocker.keys())
        now  = this.clock.now()

        locker.putPacket(packet)                      # may throw; nothing else mutated yet
        packet.markDeposited(now)
        this.codeToLocker[code] = locker

        receipt = Receipt(locker.id, code, packet.expiryTime)

    packet.notifyRecipient(receipt, this.notifier)    # I/O OUTSIDE the lock
    return receipt

LockerManager.retrievePacket(code):
    with this.lock:
        if code not in this.codeToLocker:
            throw InvalidCodeError()                  # unknown / already used

        locker = this.codeToLocker[code]
        now    = this.clock.now()

        if locker.isFree():                           # defensive: swept concurrently
            del this.codeToLocker[code]
            throw InvalidCodeError()

        # THE BUG HE ONLY FOUND WHEN HANDED THE SCENARIO:
        # expiry must be enforced HERE, not only by the periodic sweeper.
        if locker.packet.isExpired(now):
            del this.codeToLocker[code]               # burn the code immediately
            throw ExpiredCodeError(locker.packet.expiryTime)

        packet = locker.removePacket()
        packet.clearDeposit()
        del this.codeToLocker[code]
        return packet

LockerManager.reclaimExpired():
    reclaimed = []
    with this.lock:
        now = this.clock.now()
        for code, locker in list(this.codeToLocker.items()):
            if locker.isFree():
                del this.codeToLocker[code]
                continue
            if locker.packet.isExpired(now):
                p = locker.removePacket()
                p.clearDeposit()
                del this.codeToLocker[code]
                reclaimed.append(p)
    return reclaimed


# --- Tell, don't Ask ----------------------------------------------------------

Packet.notifyRecipient(receipt, notifier):
    msg = "Your package is in locker " + receipt.lockerId +
          ". Code: " + receipt.code +
          ". Collect by " + receipt.expiryTime + "."
    this.customer.notify(msg, notifier)
```

**Edge cases covered above**
- No fitting locker at all -> `NoLockerAvailableError`, no partial mutation.
- No lockers of a given size exist at all -> the scan simply finds nothing
  (a `Map<Size, List<Locker>>` with a missing key would need a guard).
- Code not in registry, or already consumed -> `InvalidCodeError`.
- Code valid but package expired and sweeper hasn't run -> `ExpiredCodeError`.
- Locker emptied by the sweeper between lookups -> stale code cleaned and rejected.
- `putPacket` on an occupied or too-small locker -> throws, even if the manager is buggy.

## 5. Design decisions

| Decision | Alternative rejected | What it gives up |
|---|---|---|
| **One public `depositPacket`** | His `getLockerForPacket` + `putPacketInLocker` | Caller can't pre-check availability without attempting a deposit. Worth it: the two-call form leaks the check-then-act window to the caller and lets an arbitrary `Locker` be passed in. |
| **`Locker` enforces fit + occupancy** | Manager-only validation | A few redundant checks. Worth it: the invariant holds even if the manager is wrong, which is the point of encapsulation. |
| **`Packet` computes its own `expiryTime`** | Manager computes it (his answer) | Packet must know `perishable` and `customer.tier`, which it already does. Keeps perishability and Prime membership out of the allocator entirely. |
| **Linear scan over `List<Locker>`** | `Map<Size, List<Locker>>` (his) | O(n) instead of O(1)-ish. Worth it at cabinet scale (tens of lockers) and it generalises: `canHold` absorbs refrigeration, security, accessibility with zero new maps. If profiling demanded it, add free-lists per size *behind* the same method. |
| **`Clock` injected** | `DateTime.now()` inline | One constructor arg. Buys the ability to unit-test the 3-day rule without waiting 3 days. |
| **`Notifier` interface** | Direct email call | An indirection. Required by the stated scope boundary and keeps I/O out of the domain. |
| **No Factory / Singleton / Builder / Strategy class** | Pattern-per-concept | Nothing. This is a ~200-line domain. **He got this right and it was the strongest column on his scorecard.** |
| **Expiry checked at retrieval AND swept** | Sweeper only (his) | A few extra lines. Without it, correctness depends on a background job's schedule. |

## 6. Extensibility — where each follow-up lands

**Refrigerated lockers.** The seam is `Locker.canHold(packet)`. Add `refrigerated: bool`
to both classes and extend one method:

```
Locker.canHold(packet):
    if not this.isFree():                              return false
    if packet.size > this.size:                        return false
    if packet.perishable and not this.refrigerated:    return false
    return true
```

`findSmallestFittingFreeLocker` is untouched. His two-map answer works for exactly this
one attribute; a third attribute (oversized, secure, ground-floor) gives him four maps
and a combinatorial fallback policy, while `canHold` absorbs each new attribute in one
line. If non-perishables should *prefer* normal lockers, that's a sort key on the scan
(`(refrigerated, size)`), not a second data structure.

**Variable hold windows.** Already on the seam: `Packet.holdDuration()`. A sixth tier
touches one method in one class. Nothing in `LockerManager` changes. *He found half of
this himself — `expiryTime` instead of `installTime` was the right instinct — then handed
the policy back to the orchestrator, which re-coupled it.*

**Multiple cabinets / locations.** `LockerManager` becomes `Cabinet`; a `LockerNetwork`
holds `Map<LocationId, Cabinet>` and routes by location. Codes become
`(locationId, code)`. Every class below `Cabinet` is untouched.

## 7. Concurrency

**Category: correctness — specifically check-then-act on shared state.** Not scarcity
(a semaphore over a locker count would still hand two drivers the same *locker*), and
not coordination (no handoff or ordering between the threads). Naming this correctly is
half the answer, and it's the half that was missing.

**Why his first answer fails.** "A lock before accessing the map" placed inside each
method does *not* fix his design, because the check (`getLockerForPacket`) and the act
(`putPacketInLocker`) are two separate public calls:

```
T_A: getLockerForPacket(pA)  -> M1 free -> returns M1     [lock taken and released]
T_B: getLockerForPacket(pB)  -> M1 free -> returns M1     [lock taken and released]
T_A: putPacketInLocker(pA, M1)                            -> M1 holds pA
T_B: putPacketInLocker(pB, M1)                            -> silently overwrites pA
```

Package A vanishes, and its code still maps to M1. His correction — "the lock must
enclose both operations" — is right, but with a two-call public API that makes locking
the *caller's* responsibility, which is not a contract you can ship.

**The smallest primitive that works: a single mutex held across the whole
find-and-place inside one public method.** That's why `depositPacket` is one call. The
lock lives on `LockerManager`, because `LockerManager` owns the collection being
mutated — `Locker` guards its own invariant but cannot see the allocation decision.

**Cost.** All deposits and retrievals serialise on one mutex — for a cabinet of tens of
lockers with human-speed traffic, contention is irrelevant, and this is the right
trade. Two things matter more than throughput: the notification I/O is deliberately
outside the lock (never hold a lock across a network call), and with only one lock
there is no lock-ordering problem, so deadlock is impossible by construction. If a
future locker network made the single mutex a bottleneck, the shard is per-cabinet —
one lock per `Cabinet`, since no operation spans two cabinets.

**Why not a lock-free CAS per locker?** `compareAndSet(null, packet)` on each candidate
with a retry loop removes the global lock, but the smallest-fit rule needs a consistent
view across all candidates to be correct — a retry loop can settle on a larger locker
than necessary. Not worth the complexity at this scale.
