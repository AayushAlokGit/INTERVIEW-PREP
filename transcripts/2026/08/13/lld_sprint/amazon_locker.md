# LLD Sprint Transcript (scoping, timeboxed)
**Date:** 2026-08-13
**Start Time:** 18:55:30 · **End Time:** 19:05:13
**Problem:** Amazon Locker system — customers pick up and return packages at a bank of physical lockers
**Category:** resource allocation
**Difficulty:** 3/5 — sizing and assignment rules must live somewhere non-obvious
**Scoping readiness: 2/5**
**Complete inside 10:00: yes** — both phases delivered by 8:32, closed voluntarily at 9:43; Out of Scope list and concurrency posture never written
**Out of Scope list produced:** Never
**Orchestrator named:** Yes (`LockerSystem`)
**Entity revision passes:** 0

| Phase | Pace target | Landed at | ± vs target | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 6:00 | 7:28 (back-fill 8:32) | +1:28 | 2 | 2/5 |
| Entities & relationships | 10:00 | 7:28 | −2:32 | 1 | 3/5 |

## Walk coverage
| # | Item | Hit/Partial/Miss | Evidence |
|---|---|---|---|
| 1 | Actors & entry point | Hit | Delivery agent deposits, customer claims via code; `LockerSystem` named as entry object |
| 2 | Core operations | Hit | Deposit, claim-by-code, reclaim after 3 days |
| 3 | Rules & legality | Hit | Same-size assignment, no free locker → exception, unknown code → exception, one package per locker |
| 4 | Lifecycle & terminal states | Partial | Deposit → claim / reclaim stated; never said what happens to a reclaimed package or whether its code is invalidated; locker states never enumerated |
| 5 | Failure behaviour | Hit | "Throw exception" used consistently for all three failure paths — one convention, held |
| 6 | Multiplicity & domain variants | Partial | Asked one-package-per-locker at 8:09 (late); never probed size upgrade (small package into large locker) or multi-site |
| 7 | Concurrency posture | Miss | Never asked whether multiple agents/customers act on the bank concurrently |
| 8 | Explicit Out of Scope | Miss | None produced; 20 seconds of budget left unused |

**Budget allocation:** 1:54 on five clarifying questions · 5:34 composing requirements + entities in one message (landed 7:28) · 0:41 on a single follow-up question (one package per locker) · 0:23 to back-fill requirement 5 · closed at 9:43 with budget remaining.

**First-pass completeness:** Requirements message two added only "one package per locker" — a multiplicity question that belonged in the opening question batch alongside the size question. Entities were single-pass and stood unchanged. No late back-fill beyond requirement 5.

**Silent assumptions:** (a) Exact size match — a small package never occupies a large locker. Never asked; costs locker utilisation and is the most likely interviewer follow-up in this domain. (b) Single locker bank at a single site. (c) Single-threaded access to the bank — two delivery agents could be assigned the same free locker. (d) The one-time code's fate on reclaim.

**Dangling rules:** Reclaim at 3 days is stated but its consequences are unresolved — where the package goes, whether the code is invalidated. Customer notification (req 2) is asserted with no mechanism or owner.

## What he produced (verbatim, as it stood at 10:00)

### Clarifying questions (1:54)
1. How will packages be deposited?
2. How will customers extract the package?
3. Are there sizes for packages?
4. How should system assign lockers to packages?
5. How long are packages allowed to occupy lockers?

### Requirements
```
0. Packages are of sizes S, M and L and similalry for lockers
1. A delivery agent deposits a apckage , system assigns it to a locker of same size. If no empty locker then cant deposit package throw exception.
2. A one time code generated when package deposited and csutomer of package notified.
3. If package not claimed within 3 days of deposit it is reclaimed and locker emptied.
4. Customer can claim package by inputting one time code ,which is code for matching locker.If unknown code entered throw exception
5. One package per locker.          [back-filled at 8:32]
```

### Out of Scope
_(none produced)_

### Entities & relationships
```
1. LockerSystem (List<Locker> , Map<one time code, locker>
2. Locker (size,package, packageDepositTime, packageExpiryTime)
3. Package (customer, size)
4. Size (S | M | L)
```

## What was still missing at 10:00
- Any Out of Scope list (hard cap: requirements max 2).
- Concurrency posture — never asked, never stated.
- Locker state model (FREE/OCCUPIED) and the terminal states of a package.
- The size-fit rule beyond exact match.
- An entity owning the code + package + expiry triple; the code is a bare map key, so invalidation-on-reclaim has no home.

## Where the time went
Well-allocated overall: a tight opening question batch, then one composed submission carrying both phases. The overrun on requirements (+1:28) is composition time, not discovery time — the structure was already there. The real cost is not overrun but under-spend: he closed the box with budget remaining and the cheapest item on the walk unwritten.

## Ideal front half (writable in the same 10 minutes)

### Requirements
**Out of Scope:** payments/billing · locker hardware & door mechanics · notification transport (SMS/email) · multi-site routing · returns/drop-off flow · package damage or theft.

1. Actors: delivery agent (deposit), customer (pickup), system job (expiry sweep). Entry point: `LockerSystem`, one bank at one site.
2. Ops: `deposit(package)` → assigns locker + issues code; `pickup(code)` → opens locker, releases it; `sweep()` → reclaims expired.
3. Rules: package of size X fits any locker of size ≥ X, smallest fit chosen; one package per locker; code is single-use and dies on pickup or reclaim.
4. Lifecycle: locker `FREE → OCCUPIED → FREE`. Package `DEPOSITED → PICKED_UP` (terminal) or `→ RECLAIMED` (terminal — locker freed, code invalidated).
5. Failure: throw on no fitting locker, unknown code, expired code. One convention.
6. Multiplicity: many lockers, three sizes, one package per locker; **a small package may occupy a large locker** — the variant this domain hinges on.
7. Concurrency: multiple agents and customers at the bank simultaneously; assignment and pickup atomic so two agents never receive the same locker.
8. (Out of Scope as stated above.)

### Entities & relationships
- `LockerSystem` — **orchestrator**; owns `List<Locker>` and `Map<code, Reservation>`
- `Locker` → size, state, current `Reservation?`
- `Reservation` → package, locker, code, depositedAt, expiresAt — **exists only to hold a rule**: requirement 3 makes the code single-use and killable on either pickup or reclaim, and requirement 4 makes expiry a terminal transition. Neither has a home when the code is a bare map key and the timestamps sit on `Locker`.
- `Package` → id, size, customerId
- `Size` enum

**What this buys — requirements:** the size-upgrade rule and the code-invalidation-on-reclaim rule, both left undecided in his version, plus the concurrency line he never asked for.
**What this buys — entities:** a `Reservation` to hold those two rules instead of scattering `packageDepositTime`/`packageExpiryTime` across `Locker` with no owner for the code.

## Feedback given
- Both phases fit the box; entities were first-pass and would need no rework. Held to 2/5 on requirements purely by the missing Out of Scope list.
- Item 5 (one failure convention) is now a genuine strength — "throw exception" applied uniformly.
- Item 7 missed entirely on a domain where two agents at one bank is the obvious contention.
- The one-package-per-locker question belonged in the opening batch, not at 8:09.
- Habit to change: write the Out of Scope list *first*, before requirement #1 — three bullets, fifteen seconds — because the end of the box never has room for it.
