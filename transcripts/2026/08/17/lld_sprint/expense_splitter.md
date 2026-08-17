# LLD Sprint Transcript (front half, timeboxed)
**Date:** 2026-08-17
**Start Time:** 18:07:21 · **End Time:** 18:32:05
**Problem:** Shared expense splitter — a group of people record expenses paid on each other's behalf and settle up
**Category:** product domain
**Difficulty:** 3/5 — one real legality rule to place (splits must sum to the total) and one genuine trade-off (stored status vs derived balance)
**Front-half readiness: 3/5**
**Complete inside 22:00: no** — class design landed at 24:44 (+2:44). Improvement on the morning's 29:46, still over.
**Out of Scope list produced:** Unprompted (durability, UI) — second consecutive session
**Orchestrator named:** Yes (`ExpenseSplitterApp`)
**Entity revision passes:** 1 (class design restated the list: `dues:Due` → `List<Dues>`, `amountPaid` added, `debtSimplificationStrategy` added, status enum changed)
**Untyped state or signatures:** `-debtSimplificationStrategy:` (no type at all) · `amount: MONEY (INT)` (a note, not a type) · `Map<User.id, amount:MONEY>` (field path used as a key type) · `simplifyDebts(List<Dues>)` (unnamed parameter, `Dues`/`Due` inconsistent) · every mutator returns `void`

| Phase | Pace target | Landed at | ± vs target | Messages used | Score |
|---|---|---|---|---|---|
| Requirements | 6:00 | 7:43 | +1:43 | 4 (two question batches, submission, one late question at 8:36) | 3/5 |
| Entities & relationships | 10:00 | 12:45 | +2:45 | 1 | 3/5 |
| Class design | 22:00 | 24:44 | +2:44 | 1 | 2/5 |

## Walk coverage
| # | Item | Hit/Partial/Miss | Evidence |
|---|---|---|---|
| 1 | Actors & entry point | Hit | "Actors → Users, Point of Contact → ExpenseSplitterSystem" — entry point named *inside requirements*, first time in this drill |
| 2 | Core operations | Hit | Create group, add expense (group and non-group), three split modes, simplify debts, settle up with partial payment |
| 3 | Rules & legality | Partial | Split modes enumerated; validation appears only as class-design comments. No rule that splits must sum to the expense total — the central legality rule of this domain |
| 4 | Lifecycle & terminal states | Partial | Asked "what is the lifecycle of an expense?", told "yours to define", never defined one. A `Due` lifecycle appeared in class design as two conflicting enums (`CLEARED/PARTIALLY CLEARED/UNCLEARED` in entities vs `OPEN/CLEARED/PARTIALLY_CLEARED/SIMPLIFIED` in classes) |
| 5 | Failure behaviour | Partial | Asked "what are some common failure modes?", told "yours to define", never named a convention. Two ad-hoc "throws exception" comments plus one bare `// validation on amount`; every method returns `void` |
| 6 | Multiplicity & domain variants | Hit | Two real probes: "are expenses always part of a group?" and "can expenses be paid for by multiple people?" — acted on the first (`nonGroupExpense`, `nonGroupDues`). The item he missed this morning |
| 7 | Concurrency posture | Miss | Never asked. Two threads settling the same `Due`, or two expenses mutating the same `dues` list — nothing. He asked this item this morning and dropped it today |
| 8 | Explicit Out of Scope | Hit | Durability, UI — unprompted, second consecutive session |

**Budget allocation:** 0:00–7:43 requirements including 8:36 of clarifying questions spread over three separate messages (a strong five-question opening batch at 3:20, a two-question batch at 6:43, one more at 8:36) · 7:43–12:45 entities (5:02) · 12:45–24:44 class design (11:59, 48% of the box).

**First-pass completeness:** One submission per phase, one revision pass, no late back-fill. The clarifying questions themselves were the best in the record — the opening batch alone covered walk items 3, 4, 5 and 6 — but arrived as three round-trips instead of one. Content of the asking is now good; packaging costs him time. Third sitting running, the constraint is composition throughput, not thinking.

**Silent assumptions:** Currency — never asked, and multi-currency is the classic hidden variant in this domain; would have cost a re-model of `amount` everywhere. Whether a user can belong to multiple groups. Whether an expense can be edited or deleted after creation (his `Due`s are derived from expenses, so an edit would invalidate every downstream due). Concurrency entirely.

**Dangling rules:** Two, both raised by his own questions and left unresolved — the **expense lifecycle** ("what is the lifecycle of an expense?" → "yours to define" → never written) and the **failure convention** ("what are some common failure modes?" → "yours to define" → never written). Caps Requirements at 3. Also `SIMPLIFIED` was added to `DueStatus` with no requirement behind it and no method that sets it.

**Untraceable state / unenforceable requirements:**
- **Requirement 3 (three split modes) is enforced by nothing.** `Expense.splitMap` arrives pre-computed from the caller; no class computes equal, exact or percentage splits, and nothing validates that shares sum to the amount. Meanwhile a strategy interface *was* built for debt simplification, which requirement 4 mentions once — the pattern went to the smaller variation point.
- **Requirement 5 (settle up) has no entry point.** `Due.recordPayment(amount)` exists and is correctly placed, but `ExpenseSplitterApp` exposes only `createExpenseGroup`, `addExpenseToGroup`, `addNonGroupExpense`. No `settleUp`, and no way for a client to obtain a `Due`.
- **Partial settlement has no record.** Modelled as `Due.amountPaid`, a running total; who paid, how much and when are unrecoverable.
- `createExpenseGroup(users) -> void` never returns the group, and `addExpenseToGroup(expense)` takes no group identifier — two signatures that cannot compose.
- `Due.status` is stored *and* derivable from `amount` vs `amountPaid`, and `getStatus()` returns the stored copy — the two can diverge.
- `User` degrades from `(id, name)` in entities to `-id:string` in class design — an actor class holding no rule.

## What he produced (verbatim, as it stood at 22:00)

### Requirements
```
1. Actors -> Users , Point of COntact -> ExpenseSplitterSystem
2. Users can create groups of members and add expenses which can be shared by other users.
   Expense can also be added outside of a group.
3. Expense sharing has 3 options -> 1. equally split b/w all people 2. exact amount split. 3. Percentage split.
4. In expense groups there should be feature to simplify the debts.
5. Users can settle up dues they have with other users in a group or otherwise. Partial due settlement allowed.
```
Clarifying questions asked, in order: are expenses always part of a group · how is the expense split between different people · is there need for simplify debts · what is the lifecycle of an expense · what are some common failure modes · is there any other functionality I am missing · can dues be settled partially · is this in memory / do we need durability · can expenses be paid for by multiple people.

### Out of Scope
```
1. Durability
2. UI
```

### Entities & relationships
```
1. User (id, name)
2. Expense (paidBy:User, amount, splitMap)
3. ExpenseGroup (users:List<User>, expense:List<Expense>, dues:Due)
4. Due (dueAmount, status: CLEARED | PARTIALLY CLEARED | UNCLEARED, debtorUser, creditorUser)
5. ExpenseSplitterApp (groups:List<ExpenseGroups>, nonGroupExpense:List<Expenses>, nonGroupDues:List<Dues>)
```

### Class design
```
class User:
-id:string

class ExpenseSplitterApp:
-expenseGroups:List<ExpenseGroups>
-nonGroupExpense:List<Expenses>
-nonGroupDues:List<Dues>
-validateExpense(expense:Expense) // throws InvalidAmount error
+createExpenseGroup(users:List<User>) -> void
+addExpenseToGroup(expense:Expense) -> void
+addNonGroupExpense(expense:Expense) -> void

class ExpenseGroup:
-users:List<User>
-expenses:List<Expense>
-dues:List<Dues>
-debtSimplificationStrategy:
-validateExpense(expense:Expense) -> void // throws exception if invalid amount of members not in group
+addExpense(expense:Expense) -> void
+simplifyDebts() -> void

interface DebtSimplificationStrategy:
+simplifyDebts(List<Dues>) -> List<Due>

class DefaultSimplificationStrategy:
+simplifyDebts(List<Dues>) -> List<Due>

class Expense:
-amount: MONEY (INT)
-paidBy:User
-splitMap:Map<User.id, amount:MONEY>
+createDues() -> List<Due>

class Due:
-amount: MONEY
-debtor:User
-creditor:User
-amountPaid: MONEY
-status: DueStatus
+getStatus() -> DueStatus
+recordPayment(amount:MONEY) -> void // validation on amount

Enum DueStatus: OPEN | CLEARED | PARTIALLY_CLEARED | SIMPLIFIED
```

## What was still missing at 22:00
An expense lifecycle · a named failure convention · any concurrency posture · the rule that splits must sum to the expense total · a `Settlement`/`Payment` entity · a `SplitStrategy` for the three split modes · a `settleUp` method anywhere on the orchestrator · a way for a client to obtain a `Due` or a `Group` · a group identifier on `addExpenseToGroup` · return types on every mutator · a type on `debtSimplificationStrategy` · currency posture.

## Where the time went
8:36 on clarifying questions across three messages — the questions were the strongest of any sprint in the record (the opening batch covered four walk items at once) but were serialised into three round-trips. Class design then took 11:59, 48% of the box, and re-typed the entity list rather than referring to it. Nothing was re-derived or rebuilt; the scaffold was being filled in.

Worth preserving: the entry point named inside requirements; two genuine domain-variant probes; `Due` reached for unprompted as the object that exists only to hold a rule (same instinct as `ShowSeat` this morning — now consistent); `Due.recordPayment` as a real Tell-Don't-Ask method with the rule sitting on the state it acts on; the second consecutive unprompted Out of Scope list.

## Ideal front half (writable in the same 22 minutes)

### Requirements (target 6:00)
1. **Actors & entry point.** One actor: `User`. Entry point `ExpenseSplitterService`. Users pre-registered.
2. **Core operations.** `addExpense(payer, amount, participants, splitSpec)` · `settleUp(from, to, amount)` · `getBalances(userId)` · `simplifyDebts(groupId)`. Groups optional — an expense names a group or a bare participant list.
3. **Rules.** (a) Exactly one payer per expense. (b) The split must **sum exactly to the expense amount** — exact splits rejected otherwise, percentages must total 100%, equal splits distribute remainder cents deterministically to the first n participants. (c) The payer may be a participant. (d) A settlement must be > 0 and ≤ the outstanding balance. (e) Money is integer minor units, never floats.
4. **Lifecycle.** `Expense`: created → immutable. Pairwise `Balance`: a signed integer, never a status. `Settlement`: immutable, append-only. Terminal state is "net balance zero" — derived, never stored.
5. **Failure convention, one, held everywhere.** Mutations throw `ExpenseException` subtypes (`SplitMismatch`, `UnknownUser`, `OverSettlement`, `NonPositiveAmount`); queries return empty collections, never null; every mutating method returns the object it created.
6. **Multiplicity & variants.** A user is in many groups; an expense belongs to at most one group; three split modes — **this is the real variation point and it gets the strategy**. Single currency; multi-currency explicitly out of scope.
7. **Concurrency posture.** Multiple threads add expenses and settle concurrently. Balances live in a `BalanceSheet` keyed by unordered user pair with **one lock per pair** — the only contended state. Cost: two settlements between the same pair serialise, everything else is parallel. Simplification takes a group-level lock.
8. **Out of Scope.** Persistence · UI · auth · notifications · multi-currency · expense edit/delete · comments/attachments · settlement reversal.

> **What this buys:** rule 3(b) is the legality rule this domain lives on and was absent from his version. Naming the failure convention once removes the `void`-plus-comment pattern from every later signature. Item 7 gives the contended state a home.

### Entities & relationships (target 10:00)
```
User            (id, name)
Expense         ──refs──> payer, participants; owns a SplitSpec    [immutable]
SplitStrategy   (Equal | Exact | Percentage)  ← computes shares
Settlement      (from, to, amount, at)                             [immutable, append-only]
BalanceSheet    ──owns──> Map<UnorderedPair<userId>, long>         ← the contended state
Group           ──owns──> members, expenseIds
ExpenseSplitterService  ← orchestrator: validation, lock acquisition, registries
```
**Entities that exist only to hold a rule:** **`BalanceSheet`** — forced by requirements 2 and 7: net debt between a pair is derived from many expenses and settlements, needs one lock per pair, and belongs to no single expense. **`Settlement`** — forced by requirement 5's partial settlements: a running `amountPaid` total cannot answer "who paid what, when." **`SplitStrategy`** — forced by requirement 6.

> **What this buys:** the two objects his version needed and lacked — one for the requirement he stated (partial settlement), one for the requirement nothing enforced (three split modes). Replacing per-`Due` status with a signed balance removes the stored-vs-derived redundancy.

### Class design (target 22:00)
```
class Expense                                   // immutable — R4
  - id: String                                  // R2
  - groupId: String | null                      // R6
  - payerId: String                             // R3(a)
  - amountMinor: long                           // R3(e)
  - shares: Map<String, Long>                   // R3(b) validated at construction
  + shares(): Map<String, Long>                                    // R5

interface SplitStrategy                         // R6 — the real variation point
  + computeShares(amountMinor: long, participants: List<String>,
                  spec: SplitSpec): Map<String, Long>              // throws SplitMismatch — R3(b)
  implementations: EqualSplit, ExactSplit, PercentageSplit

class BalanceSheet                              // owns the contended state — R7
  - net: Map<UnorderedPair, Long>               // R2 (positive = a owes b)
  - locks: Map<UnorderedPair, ReentrantLock>    // R7
  + applyExpense(e: Expense): void                                 // R2
  + recordSettlement(s: Settlement): void       // throws OverSettlement — R3(d)
  + balanceOf(userId: String): Map<String, Long>                   // R2, R5
  + pairsIn(groupId: String): List<UnorderedPair>                  // R2

class Settlement                                // immutable — R4
  - id: String  - fromUserId: String  - toUserId: String
  - amountMinor: long  - at: Instant                               // R2, R5

class Group
  - id: String  - memberIds: Set<String>  - expenseIds: List<String>    // R6

class ExpenseSplitterService                    // orchestrator: workflow only
  - users: Map<String, User>  - groups: Map<String, Group>
  - expenses: Map<String, Expense>  - settlements: List<Settlement>
  - sheet: BalanceSheet  - simplifier: DebtSimplifier               // R2
  + createGroup(memberIds: List<String>): Group                     // returns it — R5
  + addExpense(payerId: String, amountMinor: long, participants: List<String>,
               spec: SplitSpec, groupId: String | null): Expense    // throws SplitMismatch
  + settleUp(fromUserId: String, toUserId: String,
             amountMinor: long): Settlement      // throws OverSettlement — R2, R3(d)
  + balances(userId: String): Map<String, Long>                     // R2
  + simplify(groupId: String): List<Settlement>                     // R2
```
**Why each rule sits where it sits:** split validity (3b) lives in `SplitStrategy`, the object that produces the shares — validating elsewhere means trusting a caller-supplied map, which is what his version does. Over-settlement (3d) lives in `BalanceSheet` because only it knows the outstanding amount and holds the lock that makes check-then-act atomic. `Expense` and `Settlement` are immutable so nothing mutates them behind the balance sheet's back. The service validates identity, picks a strategy, delegates — and every mutation returns the object it created so the caller always has a handle.

> **What this buys:** `settleUp` exists and is reachable, `createGroup` returns the group, `addExpense` names its group — the three composition failures in his version. Every method has a return type and one exception family. Requirement 7 gets a named lock instead of never being asked.

## Feedback given
- Best clarifying questions in the record — the opening batch covered walk items 3, 4, 5 and 6 at once. But **he asked two walk questions, was told "yours to define," and then defined neither** (expense lifecycle, failure convention). That is worse than not asking: it puts the gap on the record and then walks past it. Caps Requirements at 3.
- Item 7 (concurrency) asked this morning, dropped today; item 6 missed this morning, hit today. He is trading walk items in and out rather than running all eight.
- `Due` reached for unprompted, and `Due.recordPayment` puts the rule on the state it acts on — the strongest structural instinct he shows, now consistent across two sprints.
- The strategy pattern went to the small variation point (debt simplification, mentioned once) while the large one (three split modes, requirement 3) got nothing — `splitMap` arrives pre-computed and unvalidated.
- The product's primary operation, settle up, has no entry point; `createExpenseGroup` returns `void` and `addExpenseToGroup` takes no group id, so the two cannot compose.
- The one habit to change: **when you ask a walk question and the answer is "yours to define," type the answer into the numbered requirements before you send anything else.**

**Sitting note:** problem 2 (database connection pool, 4/5, concurrency-first) was served at 18:32:05 and not attempted. Not scored.
