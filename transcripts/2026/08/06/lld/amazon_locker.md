# LLD Round Transcript
**Date:** 2026-08-06
**Start Time:** 15:13:59 · **End Time:** 16:03:10 · **Duration:** 49 min
**Problem:** Amazon Locker
**Category:** resource allocation
**Performance Rating:** 3/5  <!-- machine-read on future rounds; ≤3 = eligible for re-ask, ≥4 retired -->
**Hints Used:** 0/2
**Requirements Asked:** how the driver obtains a compartment · whether packages have sizes and whether size drives selection · one package per locker or many · whether the customer is notified · whether there is a hold time limit and what reclaim looks like · what happens if the driver has a wrong-sized package · how the customer retrieves the package (code entry) — seven unprompted questions, asked one per turn
**Never Asked:** whether the system is single- or multi-threaded · whether a driver handed a locker can walk away without using it · whether a code is single-use · whether one customer can have several packages · whether there are multiple cabinets/locations · what the error contract is (exception vs null) · how codes are generated and what happens on collision
**Out of Scope list produced:** Never
**Self-Verified:** No — submitted three methods with no trace and no edge-case list. When handed the Day-1/Day-5 scenario he found the real bug immediately (expiry enforced only by the sweeper, so a stale code works until the job runs) and his claimed trace was correct.
**Concurrency follow-up:** Correctness / check-then-act (two drivers, one free Medium locker). Named contention and "race conditions" but never named the category; never interleaved the threads despite being asked twice. First answer ("lock before accessing the map") is wrong for his two-call API; corrected to "the lock must enclose both operations" without acknowledging that this pushes locking onto the caller. No cost stated beyond "serialises the threads."
**Would it have fit a real 40-min round?** No — cut off mid-concurrency-follow-up, right at the wrong answer and before the correction.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements + Out of Scope | 5 min | 11:10 | +6:10 | No |
| Entities & relationships | 8 min | 20:29 | +12:29 | No |
| Class design | 20 min | 29:42 | +9:42 | No |
| Implementation + trace | 32 min | 36:32 | +4:32 | No |
| Extensibility + concurrency | 40 min | 49:00 | +9:00 | No |
| **Total** | 40 min | **49:00** | **+9:00** | No |

*Note: the end of the self-verification trace was not stamped; the implementation row uses the timestamp of his pseudo-code submission (15:50:31). All other stamps are exact.*

---

## Problem Statement

> **Design an Amazon Locker system.**

Given deliberately under-specified. Framing offered on request, with no rules attached:
an Amazon Locker is a physical cabinet at, say, a grocery store — a grid of individual
lockers of differing sizes. A delivery driver puts a customer's package into one of them;
the customer later comes and takes it out. Everything else (who can open what, what
happens when nothing fits, how long a package sits there, what a customer needs to prove
they're the right person) was left undecided for him to scope.

---

## Conversation Log

**Interviewer:** [15:13:59] LLD round start. Design sheet created. Five phases and reference timeline stated (measured, not enforced). Asked for language choice and noted no UML needed. "Any questions before you start?"

**Aayush:** can you describe a locker system more what all it an do /

**Interviewer:** [15:15:00] Gave the one-line physical framing only (cabinet at a store, grid of lockers of differing sizes, driver deposits, customer collects). Declined to hand over rules: "Everything else is undecided, and deciding it is part of this round. Ask me specific questions."

**Aayush:** how will the delivery driver get the locker compartment to store the packer ?

**Interviewer:** [15:16:27] The driver arrives with a package and asks the system to allocate a compartment. The system picks one and returns which compartment to use, plus a code the customer will use to open it. If nothing suitable is free, the system tells the driver and the driver takes the package away.

**Aayush:** are packages of different sizes and does packwet size matter in compartment selection?

**Interviewer:** [15:17:09] Yes to both. Packages come in Small, Medium, Large; lockers have the same three sizes. A package fits any locker of its size or bigger. When several fit, use the smallest one that works, so you don't burn a Large on a Small package.

**Aayush:** can a compartment hold multipl packages ?

**Interviewer:** [15:18:14] No. One locker holds exactly one package at a time.

**Aayush:** the customer needs to be notified of their comaprtment and code right

**Interviewer:** [15:20:06] Yes. When a package is placed, the customer is notified with the locker location and a one-time code. Treat the delivery channel (email, SMS) as someone else's problem — you just trigger the notification.

**Aayush:** is there any time limit for the customer to get back their packet before it is collected to free up space or can packet occupy lcokers indefinitely ?

**Interviewer:** [15:23:19] There's a limit: 3 days from placement. After that the package is expired and gets reclaimed — a driver comes, opens it, takes the package back, and the locker becomes free again. Assume you can read the current time when you need it.

**Aayush:** what happens when the dlievery driver tries to put a wrong sized package in the locker assigned by system ?

**Interviewer:** [15:24:29] That can't happen — the driver hands the package to the system and the system decides which locker; the driver never picks. But the system itself must not hand out a locker the package doesn't fit in, and must not hand out one that's already occupied. Those are your rules to enforce.

**Aayush:** [15:25:09 — requirements submitted]
> Requirements ->
> 1. Delivery driver inputs packet details and system assigns a locker if any and also one time code. If no locker assigned packet not deposited.
> 2. Packets sizes - S , M , L and Locker sizes - S, M , L
> 3. Packet fits any locker having locker size >= packet size.
> 4. Locker fits only 1 packet.
> 5. When packet placed in locker , customer notified with locker details and one time code to open locker.
> 6. If packet not taken by customer in 3 days , it is reclaimed and locker is emptied.

**Interviewer:** Recorded on the sheet. How do you want to start?
*(No Out of Scope list produced; not prompted for one — its absence is a graded signal. The smallest-fit rule given at 15:17 was never written into his requirements, though it did survive into his implementation.)*

**Aayush:** [15:31:57 — entities submitted]
> Entities ->
> 1. LockerManager (lockers:Locker[],
> 2. Locker (size:Size, occupied: bool, packet:Packet , installTime:datetime, oneTimeCode: int)
> 3. Packet (size: Size, customer: Customer)
> 4. Customer (name, email, phone..)
>
> Enums -> size: S | M | L

**Interviewer:** Three probes before class design. (1) `Locker` holds both `occupied: bool` and `packet: Packet` — which requirement needs both? (2) `oneTimeCode: int` sits on `Locker`. If a locker is emptied and later re-filled for a different customer, what happens to that field — does that placement still feel right? (3) Requirement 5 says the customer gets notified. Which of the four things you listed does that?

**Aayush:** [15:33:39]
> 1. just having the packet will be enough to determine whether a locker is occupied or not
> 2. oneTImeCode is better suited on packet.
> 3. The lockerManager will notify the customer

**Interviewer:** `installTime: datetime` on `Locker` — what does that record, the time the locker cabinet was installed or the time the current package went in?

**Aayush:** [15:34:28] the istallTIme should be on packet and it means the time the packet got installed on the locker.

**Interviewer:** Entities are on the sheet. Take it into class design — state plus method signatures, per class.

**Aayush:** how owudl the customer take the packet they would enter the locker details and one time code ?

**Interviewer:** [15:39:41] They enter just the one-time code at the kiosk — codes are unique across the whole cabinet, so the code alone identifies the locker. The system opens that locker; the customer takes the package; the locker becomes free. A wrong or expired code is simply rejected — no lockout, no retry limit.

**Aayush:** [15:42:01 — class design submitted]
> 1. LockerManager
> - sizeToLockerMap:
> - oneTimeCodeToLockerMap:
> - generateOneTimeCode(packet, locker) -> int
> - notifyCustomer(packet)
> + getLockerForPacket(packet) -> Locker | null
> + putPacketInLocker() -> void
> + takeoutPacketForCode(oneTimeCode) -> Packet | null
> - reclaimExpiredPackets() -> // runs periodically to claim lockers whose packets have not been claimed for 3 days
>
> 2. Locker
> - packet
> - size
> + putPacket()
> + removePacket()
>
> 3. Packet
> - customer
> - oneTimeCode: int|null
> + setOneTimeCode(oneTimeCode)
> + getCustomer()-> Customer
> + unsetOneTimeCode()
>
> 4. Customer

**Interviewer:** Four questions. (1) `sizeToLockerMap` — exact type, key and value? And `putPacketInLocker()` takes no arguments; what's the real signature? (2) `Packet` in your class design has `customer` and `oneTimeCode`, but the `size` and `installTime` you put there a minute ago are gone. Deliberate? (3) `getLockerForPacket` and `putPacketInLocker` are both public — walk me through what the driver-facing caller does with them, in order. (4) `getCustomer()` on `Packet` — who calls it, and what do they do with the result?

**Aayush:** [15:43:41]
> 1. key - SizeEnum Value list of locker objects.
> 2. Forgot to list those fields
> 3. the driver will first try to get a locker for packet if no locker gotten skip else put packet in locker.
> 4. The lockerManager will call the getCustomer on packet and they will use it to notify teh cusotmer

**Interviewer:** One clarification for the sheet: does the list under each size hold **all** lockers of that size, or only the free ones?

**Aayush:** all lockers are held

**Interviewer:** Noted. Go ahead — and tell me the language you're writing in.

**Aayush:** pseudocode

**Interviewer:** Pseudo-code it is. Write it. *(Requested `getLockerForPacket`, `putPacketInLocker`, `takeoutPacketForCode` at minimum.)*

**Aayush:** [15:50:31 — implementation submitted]
> getLockerForPacket(packet)->
> start iterating through lockers of size = packet size if empty locker found return locker.
> Else move up a size (while size <= L) and repeat.
> Return null if no empty locker found.
>
> putPacketInLocker(packet, locker)->
> 1. generate a one time code for packet + locker combo
> 2. save oneTimeCode to locker in map of LockManager
> 3. Update install time and one time code in packet an put in locker.
> 4. notify customer.
>
> takeoutPacketForCode(oneTimeCode)->
> Check if oneTimeCode exists in map if it does nto exist tehn throw an exception for wrong code.
> Check locker if the locker does not have a packet throw an exception.
> Remove packet from locker
> Unset oneTImeCode and installTIme in packet.
> remove oneTimeCode and locker mapping form the map

**Interviewer:** *(No trace offered, no edge cases enumerated — had to ask.)* Two things. (1) Your signature says `takeoutPacketForCode(oneTimeCode): Packet | null`, but the body throws. Which is it? (2) Trace this concretely, naming the state after each step: cabinet has 2 Small lockers and 1 Medium. S1 and S2 are both occupied. A driver arrives with a Small package for Alice on Day 1. Then Alice comes to the kiosk on Day 5, and the periodic reclaim job has not run since Day 1. Walk it.

**Aayush:**
> 1. let the body throw instead of null
> 2. alice will still find here package since periodic reclaim job has not yet run , we can add a check when trying to extrct the package to check for the installTIme eing older tah 3 days

*(Claimed trace verified silently — correct. The bug is real: expiry was enforced only by the sweeper.)*

**Interviewer:** Good — record that fix. **Follow-up 1.** We're adding refrigerated lockers. Some packages are perishable and must go in a refrigerated locker; non-perishable packages may go in either. Size rules unchanged. What in your design changes?

**Aayush:** [15:55:32] we would keep 2 maps now one for refrigerated locker by sizes and another for normal lockers . For perishable packets always pick from refrgerate dlockers map when assigning a locker. And for non perishable furst check in normal locker map if nothing found then check refrigerated locker map

**Interviewer:** **Follow-up 2.** The 3-day window is no longer fixed. Perishable packages hold for 1 day, standard for 3, and Prime customers get 5. Where does that number live in your design now, and who decides it?

**Aayush:** [15:56:10] the number will live in Packet still but this tie we will store the expiryTime instead of installTIme

**Interviewer:** Who computes that `expiryTime` — i.e. which object holds the rule "perishable ⇒ 1 day, Prime ⇒ 5, otherwise 3"?

**Aayush:** [15:56:32] the lockerManager

**Interviewer:** **Follow-up 3 — concurrency.** The cabinet now serves multiple drivers at once; the kiosk runs one thread per driver against the same `LockerManager`. Exactly one Medium locker is free. Driver A and Driver B each arrive with a Medium package at the same instant. What breaks?

**Aayush:** [15:59:48] now with cntention the common resource that is the locker map needs to be protected, in order to avoid race conditions.

**Interviewer:** Concretely: which two operations, in which order, produce the wrong outcome — and what is the wrong outcome? Then tell me what you'd do about it and what that costs.

**Aayush:** could you larify your questiuo or reframe it

**Interviewer:** [16:00:25] Three parts. (1) Interleave the two threads' calls into your existing methods, step by step, and tell me what ends up in that Medium locker. (2) What's the smallest thing you'd add to prevent it? (3) What does that cost you?

**Aayush:** we need to add a lock before accessing the map. This woudl serialsie the threads

**Interviewer:** [16:01:28] Your driver flow is two separate public calls — `getLockerForPacket`, then `putPacketInLocker`. Given that, where exactly does the lock go, and does putting it "before accessing the map" actually stop A and B both being handed that Medium locker?

**Aayush:** [16:02:58] the lock must enclose both the operations. Yes putting lock before accessing the map ensures that only one thread is handed the locker and only 1 packet is assigned

**Interviewer:** [16:03:10] End of round. Feedback delivered.

---

## His Design

**Requirements he gathered:**
1. Delivery driver inputs packet details and system assigns a locker if any and also one time code. If no locker assigned packet not deposited.
2. Packet sizes — S, M, L and Locker sizes — S, M, L.
3. Packet fits any locker having locker size >= packet size.
4. Locker fits only 1 packet.
5. When packet placed in locker, customer notified with locker details and one time code to open locker.
6. If packet not taken by customer in 3 days, it is reclaimed and locker is emptied.
7. *(added later, during class design)* Customer extracts package using one time code which uniquely identifies a locker in the system.

**Out of Scope:** None. Never produced, never prompted.

**Entities & relationships:**
- **LockerManager** — `lockers: Locker[]`. Orchestrator; notifies the customer.
- **Locker** — `size: Size`, `packet: Packet`. (`occupied: bool` dropped when challenged — presence of `packet` determines occupancy.)
- **Packet** — `size: Size`, `customer: Customer`, `oneTimeCode: int`, `installTime: datetime`. (`oneTimeCode` and `installTime` both moved here from `Locker` when challenged.)
- **Customer** — `name`, `email`, `phone`, ...
- **Enum** `Size: S | M | L`

**Class outlines:**
```
LockerManager
- sizeToLockerMap: Map<Size, List<Locker>>   // holds ALL lockers of that size, not only free ones
- oneTimeCodeToLockerMap: Map<int, Locker>
- generateOneTimeCode(packet, locker): int
- notifyCustomer(packet)
- reclaimExpiredPackets()          // runs periodically
+ getLockerForPacket(packet): Locker | null
+ putPacketInLocker(): void        // real signature clarified as (packet, locker)
+ takeoutPacketForCode(oneTimeCode): Packet | null   // later: throws instead

Locker
- packet
- size
+ putPacket()
+ removePacket()

Packet
- customer
- oneTimeCode: int | null
- size            // stated in entities, omitted from class list by oversight
- installTime     // stated in entities, omitted from class list by oversight
+ setOneTimeCode(oneTimeCode)
+ unsetOneTimeCode()
+ getCustomer(): Customer

Customer
(no members stated)
```
Caller flow (driver): call `getLockerForPacket`; if null, skip; else call `putPacketInLocker`.

**Core implementation:**
```pseudocode
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

**Revisions after the prompted trace:**
- `takeoutPacketForCode` throws on error rather than returning null.
- Add a check in `takeoutPacketForCode` for `installTime` being older than 3 days, so an expired code is rejected even when the periodic reclaim job has not yet run.

**Follow-up answers:**
1. *Refrigerated lockers* — two maps, one refrigerated-by-size and one normal-by-size. Perishable always from the refrigerated map; non-perishable checks normal first, falls back to refrigerated.
2. *Variable hold windows* — keep the number in `Packet` but store `expiryTime` instead of `installTime`; `LockerManager` computes it.
3. *Concurrency* — protect the shared locker map with a lock to avoid race conditions; the lock must enclose both `getLockerForPacket` and `putPacketInLocker`.

**Gaps / misplaced responsibilities:**
- **No Out of Scope list at any point.** Hard ceiling at 3/5.
- **Two-call public API** (`getLockerForPacket` + `putPacketInLocker`) leaks the check-then-act window to the caller and accepts an arbitrary `Locker` from outside. This is the root cause of both the trust bug and the concurrency race.
- **`Locker` enforces no invariant.** `putPacket()` checks neither occupancy nor fit. The one rule `Locker` exists to own — "this package fits in me and I am free" — lives only in `LockerManager` by construction.
- **`getCustomer()` on `Packet` is a textbook Ask** — the manager reaches into the packet to pull out a customer to notify, instead of telling the packet to notify its recipient.
- **Hold-window policy placed in `LockerManager`** on follow-up 2, coupling the allocator to perishability and Prime membership. He had the right instinct (`expiryTime` as data on the packet) and then handed the rule back to the orchestrator.
- **Expiry originally enforced only by the sweeper** — found only when handed the scenario.
- **Signature contradicted body** (`Packet | null` vs. throwing); two `Packet` fields silently dropped from the class list.
- **`reclaimExpiredPackets` never implemented**, and after the switch to `expiryTime` the retrieval check was still described against `installTime`.
- **Refrigeration handled by duplicating the data structure** rather than generalising the fit predicate — does not survive a third attribute.
- **Concurrency category never named** (correctness / check-then-act); threads never interleaved despite two requests; no cost stated beyond "serialises the threads"; never acknowledged that "lock enclosing both operations" makes locking the caller's job under his API.

---

## Feedback Given

### Round Conditions

**Hints used: 0/2.** No ceiling from hints — everything on the sheet is his. That's genuinely good; most people burn one asking "should there be an X class?"

**Requirements asked (unprompted):** how the driver obtains a compartment · whether packages have sizes and whether size drives selection · one package per locker or many · whether the customer gets notified · whether there's a hold time limit and what reclaim looks like · what happens if the driver has the wrong-sized package · how the customer retrieves (code entry). Seven questions before writing anything. Strong.

**Never asked:** whether a code is single-use or reusable · whether one customer can have several packages at once · whether there are multiple cabinets/locations · whether the system is single- or multi-threaded · what the error contract is (exception vs. null) · whether a driver who's been handed a locker can walk away without using it. That last one is the expensive miss — never asking it produced the two-call public API, and that hole is exactly what the concurrency follow-up walked through.

**Out of Scope list: never produced.** Not prompted, not volunteered. Hard ceiling.

**Self-verification: No.** Three methods submitted, then stop. Had to be handed the Day-1/Day-5 scenario. To his credit, once handed it he found the real bug immediately and correctly — expiry enforced only by the sweeper, so a stale code works until the job runs. His claimed result matched the real trace. But he didn't go looking.

### Rubric

**Requirements & scoping — 2/5.** Seven precise, well-ordered questions would have been a 4. No Out of Scope list caps it here, and he never wrote down the smallest-fit rule he was given even though he implemented it.

**Entity modelling — 4/5.** Four entities, right granularity, orchestrator named without being asked. Collapsed `occupied: bool` the moment it was questioned and re-homed `oneTimeCode` and `installTime` from `Locker` to `Packet` — both correct calls, made fast.

**Class design — 3/5.** Signatures incomplete on submission (`putPacketInLocker()` with no args, two fields silently dropped from `Packet`), and the return type contradicted the body — `Packet | null` against a method that throws. Fixed when asked, but a senior submits a consistent contract the first time.

**Responsibility placement — 2/5.** Weakest column. `Locker.putPacket()` accepts anything: doesn't check that it's empty, doesn't check that the package fits. The one rule `Locker` exists to own lives nowhere, so `LockerManager` enforces it by construction and nothing enforces it if anyone calls `putPacket` directly. `getCustomer()` on `Packet` is a textbook Ask. And on follow-up 2 the hold-window rule went into `LockerManager` too. The manager is accreting rules that belong to the objects holding the state.

**Implementation & correctness — 2/5.** `getLockerForPacket` is correct — smallest-fit falls out of starting at the packet's size and walking up. `takeoutPacketForCode` had a real correctness bug he didn't look for. And `putPacketInLocker(packet, locker)` validates nothing: it trusts a `Locker` handed in by an untrusted caller. Two callers, one locker — that's not a concurrency bug, that's a single-threaded bug the API allows.

**Simplicity & judgement — 4/5.** No pattern worship, no Factory, no Singleton, no speculative abstraction. The maps are the right data structure for the lookups needed. Real strength.

**Extensibility — 3/5.** Refrigerated lockers: "two maps, one per kind" works, but it's a copy, not a seam. Add a third attribute — oversized, secure, ground-floor-accessible — and it's four maps and a combinatorial fallback policy. Follow-up 2 was better: moving `installTime` → `expiryTime` is exactly right, because it turns a *policy* into *data on the object*. Then he handed the policy back to `LockerManager`, which undoes half the win.

**Concurrency — 2/5.** Named contention and race conditions, which is the right neighbourhood, but never named the **category** — check-then-act correctness, not scarcity — and never interleaved the two threads despite two requests. His first answer, "a lock before accessing the map," is wrong for his design: a lock inside each method still lets A and B both receive the Medium locker, because the check and the act are in two different public calls. He corrected to "the lock must enclose both operations," which is right, but that means the *caller* must hold a lock across two public methods — and he never acknowledged that his API forces that. No cost stated beyond "serialises the threads."

**Communication — 4/5.** Clear, compact, responsive. Changed his mind fast when a question exposed something, without getting defensive. Reasoning always visible.

### Pace Report

| Phase | Reference | Actual (cumulative) | Delta | On pace? |
|---|---|---|---|---|
| Requirements + Out of Scope | 5 min | 11:10 | +6:10 | No |
| Entities & relationships | 8 min | 20:29 | +12:29 | No |
| Class design | 20 min | 29:42 | +9:42 | No |
| Implementation + trace | 32 min | 36:32 | +4:32 | No |
| Extensibility + concurrency | 40 min | 49:00 | +9:00 | No |
| **Total** | 40 min | **49:00** | **+9:00** | No |

**Would this have fit a real 40-minute round? No.** He'd have been cut off in the middle of the concurrency follow-up — around the "lock before accessing the map" answer, before the correction that made it right. The interviewer's last impression would have been a wrong concurrency fix.

**The single biggest time sink is requirements — 11 minutes, one question per turn.** Seven separate round-trips for seven questions. Batching four at a time gets him back inside 40 on its own. Entities cost another 3 minutes of rework that a written Out of Scope list would have front-loaded. Notably, implementation was his *fastest* phase relative to reference — his coding isn't slow, his scoping is.

### Senior Readiness Debrief

**Senior-signal scorecard**

| Signal | Status | Why |
|---|---|---|
| Scopes before designing | Mixed | Excellent questions, but no Out of Scope list and no numbered confirmation before designing |
| State derived from requirements | Strong | Every field traced to a requirement; killed `occupied` instantly when challenged |
| Rules live with their state (Tell, Don't Ask) | Weak | `Locker` enforces nothing; `getCustomer()` is an Ask; hold-window policy landed in the orchestrator |
| Simplicity held under pressure | Strong | Zero patterns forced across three escalating follow-ups |
| Verifies own logic | Weak | Submitted three methods with no trace and no edge-case list; needed the scenario handed to him |
| Extends without rewriting | Mixed | Follow-up 2 landed on a seam; follow-up 1 duplicated a structure instead of generalising |

**Overall read: solid mid-level. Hire at mid, not yet senior.** His instincts about *what* to build are good and his restraint is better than most. What's missing is the senior habit of making objects defend their own invariants, and the reflex to attack his own code before someone else does.

**What a senior strong-hire would have done on THIS problem**

*The questions he skipped, and what they'd have changed.* "Is this single-threaded?" — asked at minute two, it either lands concurrency in Out of Scope (and he's safe when it's sprung) or it tells him to design the reservation atomically from the start. "Can a driver be handed a locker and not use it?" — the answer forces a single `depositPacket(packet) → Receipt` entry point instead of the two-call API, and that single change kills both the untrusted-locker bug and the entire concurrency race. One question, two bugs.

*The rule in the wrong place.* `Locker.putPacket(packet)` should be the thing that refuses:
```
Locker.putPacket(packet):
    if this.packet != null: throw LockerOccupiedError
    if packet.size > this.size: throw PacketDoesNotFitError
    this.packet = packet
```
Right now `LockerManager` is the only reason a package never lands in a locker it doesn't fit. `Locker` is the object that knows its size and its occupancy — it should be impossible to put a Large package in a Small locker even if the manager has a bug.

*The getter that should have been a behavior.* `packet.getCustomer()` so the manager can notify. Invert it: `packet.notifyRecipient(receipt, notifier)`, or give `Customer` a `notify(message)` and let the packet tell its own customer. The manager shouldn't be pulling a customer out of a packet to decide what to do with it.

*The follow-up 2 answer, finished properly.* He got 90% — `expiryTime` on `Packet` is exactly right. The last 10%: `Packet` should compute its own expiry at construction from what it already knows (`perishable`, `customer.tier`) and expose `packet.isExpired(now): bool`. Then `takeoutPacketForCode` and `reclaimExpiredPackets` both just ask the packet, and adding a sixth tier touches one class. Putting the rule in `LockerManager` means the manager now knows about perishability *and* Prime membership — two domains with nothing to do with allocating lockers.

*Where no pattern is the right answer* — and he got this right. Nobody needed a Factory for `Packet`, a Singleton for `LockerManager`, or a Strategy for locker selection. Smallest-fit is one loop. That instinct is rarer than it looks.

*The one place a pattern would genuinely have paid* — follow-up 1. Instead of a map per locker kind, a predicate: `findLocker(packet)` filters `lockers` by `locker.canHold(packet)` where `Locker.canHold` checks size *and* refrigeration *and* whatever comes next, then picks the smallest match. That's a Strategy in everything but name, it absorbs the third and fourth attribute for free, and it's fewer lines than two maps.

*The edge case his trace would have caught.* Beyond expiry: what does `getLockerForPacket` do when `sizeToLockerMap` has no entry at all for size L because this cabinet has no Large lockers? His pseudo-code iterates "lockers of size = packet size" and then moves up — with a missing key, that's a null iteration, not an empty one.

**One concrete drill.** Trace before being asked, every time — and make it the trace that hurts. After the last line of any method, before saying "done," pick the scenario most likely to break it and walk it out loud naming state after each step. Not the happy path. For `takeoutPacketForCode`, the happy path is "valid code, package there." The hurting path is "valid code, four days old" — the one he was handed, that he'd have found himself in thirty seconds. Concretely, for the next two weeks: on every LLD method, list three edge cases *in writing* before submitting — one invalid input, one illegal state, one time/boundary case. Three lines. It's the difference between a 2 and a 4 in the correctness column.

Pointed to the pre-round checklist in `lld_senior_guidance.md`, particularly the last four boxes.

### **Performance Rating: 3/5**

Solid mid-level design with good instincts and real restraint. This would have been a 3 regardless — but the binding ceiling is stated: **no Out of Scope list caps it at 3 on its own.** Zero hints used, which is the best thing on the scorecard.

---

*Full design sheet, including the complete optimal reference design, is in `amazon_locker_design.md` alongside this transcript.*
