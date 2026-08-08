# Expense Splitter (Splitwise) — Design Sheet

## 1. Requirements

1. User can create an expense group and add their friends to it.
2. User can add expenses to the group and decide how the expense should be shared and among whom it should be shared.
3. Users should be able to simplify the debts to finally get what amount they owe and to whom.
4. Users should be able to mark their dues as settled in the app.

## Out of Scope

1. Payment integration
2. Notifications

## Clarifications asked
- "Do I need multi-threaded concerns, or is this a single-threaded in-memory app?" → single-threaded, in-memory.

## 2. Entities & Relationships

- **User** (id, name, email)
- **ExpenseGroup** (id, simplifyDebtsOn, members: User[], expenses: [], debtSimplificationStrategy)
- **Expense** (postedBy: User, amount, createdAt, sharingDict: { userId : share })
  - revised: `sharingDict` holds the **actual amount** each person needs to pay; the client converts
    percentages to actual amounts before saving to the backend.
- **UserManager** (users: User[])
- **ExpenseGroupManager** (expenseGroups: ExpenseGroup[])
- **Due** — held by ExpenseGroup. Each due records who owes whom how much, and has a **state**.
  - Dues are the input to the `debtSimplificationStrategy`, which returns simplified dues for the group.
  - Marking settled: the due's state is altered to SETTLED.
- **ExpenseAppOrchestrator.** (unnamed) — holds the UserManager and the ExpenseGroupManager, and exposes methods
  to get an expense group, create new users, etc. A caller wanting to add an expense goes through this
  class to obtain the ExpenseGroup, then calls the group's addExpense method.

Revised: `simplifyDebtsOn` dropped — `debtSimplificationStrategy` alone is enough.

## 3. Class Outlines

```text
class User
    id
    name
    email

    getId()
    getName()
    getEmail()


class Expense
    id
    postedBy
    amount
    createdAt
    sharingDict

    getAmount()
    getPostedBy()
    getShares()


class Due
    debtor
    creditor
    amount
    state

    settle()
        if state == SETTLED
            error
        state = SETTLED

    isSettled()
    getDebtor()
    getCreditor()
    getAmount()


class ExpenseGroup
    id
    members
    expenses
    dues
    debtSimplificationStrategy

    addMember(user)
    removeMember(user)

    addExpense(expense)
        validateExpense(expense)
        expenses.add(expense)

        create dues from expense
        dues.add(created dues)

    simplifyDebts()
        simplifiedDues = debtSimplificationStrategy.simplify(dues)
        dues = simplifiedDues

    settleDue(due)
        verify due belongs to this group
        due.settle()

    getExpenses()
    getMembers()
    getDues()


interface DebtSimplificationStrategy
    simplify(dues) -> List<Due>


class DefaultDebtSimplificationStrategy
    simplify(dues)
        ...
        return simplifiedDues


class UserManager
    users

    createUser(name, email) -> User
    getUser(userId) -> User


class ExpenseGroupManager
    expenseGroups

    createGroup() -> ExpenseGroup
    getGroup(groupId) -> ExpenseGroup


class ExpenseAppOrchestrator
    userManager
    expenseGroupManager

    createUser(name, email) -> User
    createExpenseGroup() -> ExpenseGroup

    getUser(userId) -> User
    getExpenseGroup(groupId) -> ExpenseGroup
```

Typical flow:

```text
app.createUser(...)
app.createExpenseGroup(...)

group = app.getExpenseGroup(groupId)

group.addMember(user)
group.addExpense(expense)

group.simplifyDebts()

group.settleDue(due)
```

### Refinements given under probing

- `validateExpense` must validate that the breakdown sums up to the whole, and that the amount is
  shared between group members.
- `simplifyDebts` should **add** new simplified debts and mark the old debts' status as SIMPLIFIED
  (rather than replacing the list wholesale).
- The group must maintain a list of dues (answer to "how is a caller-supplied Due verified").
- The **Expense** object should be responsible for generating the dues, since it has access to all the
  information — sharing dict, amount, members.

## 4. Core Implementation

Signature as stated: `generateDues` takes nothing, is called on the Expense object, returns a list of Dues.

```text
generateDues()

    // 1. Validate expense amount
    if amount <= 0
        throw InvalidExpenseException

    // 2. Validate that every participant is a group member
    for each userId in sharingDict.keys()
        if userId not in groupMembers
            throw InvalidExpenseException(
                "Expense can only be shared with group members"
            )

    // 3. Validate that the breakdown sums to the full expense
    totalShared = 0
    for each userId, shareAmount in sharingDict
        if shareAmount < 0
            throw InvalidExpenseException
        totalShared += shareAmount

    if totalShared != amount
        throw InvalidExpenseException(
            "Expense breakdown does not sum to total amount"
        )

    // 4. Generate dues
    dues = []
    for each userId, shareAmount in sharingDict
        if shareAmount == 0
            continue
        if userId == postedBy.id
            continue

        debtor = findUser(userId)
        creditor = postedBy

        due = Due(
            debtor = debtor,
            creditor = creditor,
            amount = shareAmount,
            state = ACTIVE
        )
        dues.add(due)

    return dues
```

```text
class DefaultDebtSimplificationStrategy

    simplify(dues)

        // Only ACTIVE dues participate in simplification.
        activeDues = []
        for each due in dues
            if due.state == ACTIVE
                activeDues.add(due)

        // Calculate net balance for every user.
        // Positive balance -> user should receive money
        // Negative balance -> user owes money
        balance = Map<User, Amount>()
        for each due in activeDues
            balance[due.creditor] += due.amount
            balance[due.debtor]   -= due.amount

        creditors = []
        debtors   = []
        for each user, amount in balance
            if amount > 0
                creditors.add( (user, amount) )
            else if amount < 0
                debtors.add( (user, -amount) )

        simplifiedDues = []
        creditorIndex = 0
        debtorIndex   = 0

        while creditorIndex < creditors.size() AND debtorIndex < debtors.size()
            creditor = creditors[creditorIndex]
            debtor   = debtors[debtorIndex]

            settlementAmount = min(creditor.amount, debtor.amount)

            simplifiedDue = Due(
                debtor   = debtor.user,
                creditor = creditor.user,
                amount   = settlementAmount,
                state    = ACTIVE
            )
            simplifiedDues.add(simplifiedDue)

            creditor.amount -= settlementAmount
            debtor.amount   -= settlementAmount

            if creditor.amount == 0
                creditorIndex++
            if debtor.amount == 0
                debtorIndex++

        // The original dues are not deleted.
        // They are preserved for history/audit purposes.
        for each due in activeDues
            due.state = SIMPLIFIED

        return simplifiedDues
```

### Self-verification trace (prompted, not volunteered)

Scenario: members A, B, C. (1) A pays $90 dinner {A:30, B:30, C:30}. (2) B pays $60 taxi {A:20, B:20, C:20}.
(3) simplifyDebts(). (4) C settles.

His stated result:
1. `dues -> B->A 30 (not settled), C->A 30 (not settled)`
2. `dues -> B->A 30 X, C->A 30 X, A->B 20 X, C->B 20 X`
3. `dues -> B->A 30 SIMPLIFIED, C->A 30 SIMPLIFIED, A->B 20 SIMPLIFIED, C->B 20 SIMPLIFIED, C->A 40 X, C->B 10 X`
4. same, but C's dues are in SETTLED state

Unprompted observation he made before tracing: "with this design the expenseGroup will keep accumulating
dues which have been simplified as well as settled."

## 5. Follow-ups

**Multiple payers on one expense** ($200 hotel, A paid $150, B paid $50, split 3 ways):
> Expense class gets updated: `postedBy` -> `paidBy: { UserId: amount }`. The generateDues method will now
> need to generate dues considering the contribution of multiple payers, so the debtor's net payment will be
> split on the percentage of payment made by the user who actually paid. The payment split of multiple payers
> is considered for determining the dues.

**Partial settlement** (C owes A $40, pays back $25):
> The Due class will own the logic to partially settle a due; it will record each settlement effort. And now
> have an extra state called PARTIALLY_SETTLED.

**Concurrency** (T1 `settleDue`, T2 `simplifyDebts`, interleaved):
> 1. This is a correctness problem — if T1 marks a debt as settled while T2 is still processing, then the
>    simplified debt added by the simplification would be wrong as it would not account for the settled debt.
> 2. Category: correctness.
> 3. Fixed by using a lock before simplifying debts to allow no other thread to push new dues or modify
>    existing dues of the expense group.
> Cost: "lower throughput for adding dues, but this is fine for a small scale app. Reduced throughput for
> settling and adding new dues in exchange for correctness of debt simplification."

---
---

# Optimal Reference (what a senior strong-hire would design)

*Added after the round. Your own design above is untouched. This deliberately includes everything the
round withheld.*

## 1. Requirements + Out of Scope

**Functional**
1. Create a group; add/remove members.
2. Record an expense in a group: one or more **payers** (who put money in) and a **split** among
   participants (who consumed it). Split types: EQUAL, EXACT, PERCENTAGE, SHARES.
3. Query the group's outstanding balances: who owes whom, how much.
4. Simplify debts — minimise the number of transfers while preserving every net balance.
5. Settle a due, in full or **partially**, recording each payment.

**Out of scope:** payment rails, notifications, auth/permissions, persistence, multi-currency FX rates,
friend graph outside groups, expense edit/delete history.

**Questions he never asked, and what each would have changed**

| Question | Answer | Design consequence |
|---|---|---|
| Can one expense have multiple payers? | Yes | `paidBy: Map<UserId, Money>`, not a scalar `postedBy` — decided in phase 1, not phase 5 |
| Can a due be settled partially? | Yes | Balances are numeric, so partial payment needs no new state at all |
| One currency per group, or many? | One per group | `Money(amount: long minorUnits, currency)` — **integer minor units, never floats** |
| Are split types first-class? | Yes | `SplitStrategy` interface — the one place a pattern genuinely pays |
| What happens to dues when a member leaves? | Blocked while they have a non-zero net balance | `removeMember` is a guarded operation, not a list removal |
| Do settled/simplified dues stay forever? | No — collapse them | Balances are **derived**, not accumulated (see section 5) |

## 2. Entities & relationships

```
ExpenseAppService  (facade / entry point)
   +-- UserRepository        : id -> User
   +-- GroupRepository       : id -> ExpenseGroup

ExpenseGroup  (the aggregate root — owns every invariant about money in this group)
   +-- members     : Set<UserId>
   +-- expenses    : List<Expense>        (immutable facts)
   +-- settlements : List<Settlement>     (immutable facts)
   +-- ledger      : BalanceLedger        (derived, authoritative for "who owes whom")

Expense (immutable value)
   +-- paidBy : Map<UserId, Money>
   +-- owedBy : Map<UserId, Money>        (produced by a SplitStrategy at construction)

SplitStrategy (interface)   EqualSplit | ExactSplit | PercentageSplit | ShareSplit

DebtSimplifier (interface)  GreedyDebtSimplifier   -> List<Transfer>
```

**Orchestrator: `ExpenseGroup`.** `ExpenseAppService` is a thin facade that resolves ids to objects —
it contains **zero** money rules.

**The key modelling decision, and the one his design missed:** an `Expense` and a `Settlement` are
**immutable facts**; a "due" is a **derived view** over them. Storing mutable `Due` rows and then
inventing `SIMPLIFIED` / `SETTLED` states to stop them double-counting is the source of every wart in
his version — unbounded growth, the history-vs-replacement contradiction, and the strategy mutating
its own input.

## 3. Class outlines

```text
class Money                                   // value object, integer minor units
    - amount   : long          // cents. NEVER a float.
    - currency : Currency
    + plus(Money): Money   + minus(Money): Money   + negate(): Money
    + isZero(): bool       + isPositive(): bool    + min(Money): Money

class User
    - id: UserId   - name: String   - email: String
    + id(): UserId

interface SplitStrategy
    + computeShares(total: Money, participants: List<UserId>): Map<UserId, Money>
      // postcondition: shares sum EXACTLY to total (remainder distributed deterministically)

class EqualSplit       implements SplitStrategy
class ExactSplit       implements SplitStrategy   - amounts: Map<UserId, Money>
class PercentageSplit  implements SplitStrategy   - percents: Map<UserId, int>   // basis points
class ShareSplit       implements SplitStrategy   - shares: Map<UserId, int>

class Expense                                  // immutable
    - id: ExpenseId
    - description: String
    - total: Money
    - paidBy: Map<UserId, Money>
    - owedBy: Map<UserId, Money>               // resolved at construction, never recomputed
    - createdAt: Instant
    + netEffect(): Map<UserId, Money>          // paid - owed, per user; sums to zero
    + participants(): Set<UserId>

class Settlement                               // immutable
    - id: SettlementId
    - from: UserId   - to: UserId   - amount: Money   - at: Instant
    + netEffect(): Map<UserId, Money>          // +amount to `from`, -amount to `to`

class BalanceLedger                            // net position per user; single source of truth
    - net: Map<UserId, Money>                  // invariant: values always sum to zero
    + apply(effect: Map<UserId, Money>): void
    + netOf(user: UserId): Money
    + creditors(): List<Pair<UserId, Money>>   // net > 0
    + debtors():   List<Pair<UserId, Money>>   // net < 0
    + isSettled(user: UserId): bool

interface DebtSimplifier
    + simplify(ledger: BalanceLedger): List<Transfer>      // pure — reads, never mutates

class GreedyDebtSimplifier implements DebtSimplifier

class Transfer                                 // immutable suggestion, NOT persisted state
    - from: UserId   - to: UserId   - amount: Money

class ExpenseGroup
    - id: GroupId
    - currency: Currency
    - members: Set<UserId>
    - expenses: List<Expense>
    - settlements: List<Settlement>
    - ledger: BalanceLedger
    - simplifier: DebtSimplifier

    + addMember(userId: UserId): void
    + removeMember(userId: UserId): void       // throws if ledger.netOf(userId) != 0
    + addExpense(desc, total, paidBy, participants, split): ExpenseId
    + recordSettlement(from: UserId, to: UserId, amount: Money): SettlementId
    + outstandingTransfers(): List<Transfer>   // derived on demand — nothing stored
    + balanceOf(userId: UserId): Money

class ExpenseAppService
    - users: UserRepository   - groups: GroupRepository
    + createUser(name, email): UserId
    + createGroup(name, currency, ownerId): GroupId
    + addExpense(groupId, ...): ExpenseId      // resolves ids, delegates. No money rules here.
```

## 4. Core method implementations

```text
ExpenseGroup.addExpense(desc, total, paidBy, participants, split) -> ExpenseId

    // ---- validation: every rule lives here, because ExpenseGroup owns members + currency ----
    if not total.isPositive()
        throw InvalidExpense("total must be positive")
    if total.currency != this.currency
        throw InvalidExpense("currency mismatch")
    if paidBy.isEmpty() or participants.isEmpty()
        throw InvalidExpense("need at least one payer and one participant")

    for userId in paidBy.keys() + participants
        if userId not in members                       // <-- the check his Expense could not perform
            throw InvalidExpense(userId + " is not a member of this group")

    paidTotal = sum(paidBy.values())
    if paidTotal != total
        throw InvalidExpense("payer contributions must sum to the total")

    // ---- delegate the split to the strategy; it guarantees exact summation ----
    owedBy = split.computeShares(total, participants)   // throws on bad percentages/exact amounts

    expense = new Expense(newId(), desc, total, paidBy, owedBy, now())

    expenses.add(expense)
    ledger.apply(expense.netEffect())                   // ledger is the ONLY mutable money state
    return expense.id


Expense.netEffect() -> Map<UserId, Money>
    effect = {}
    for userId, paid in paidBy      : effect[userId] += paid       // put money in -> credit
    for userId, owed in owedBy      : effect[userId] -= owed       // consumed     -> debit
    // invariant: sum(effect.values()) == 0, since sum(paidBy) == total == sum(owedBy)
    return effect


EqualSplit.computeShares(total, participants) -> Map<UserId, Money>
    n = participants.size()
    if n == 0 throw InvalidSplit
    base      = total.amount / n            // integer division on MINOR UNITS
    remainder = total.amount % n            // e.g. $10 / 3 -> 333, 333, 334
    shares = {}
    sortedParticipants = participants.sortedById()    // deterministic, so it is reproducible
    for i, userId in sortedParticipants
        cents = base + (1 if i < remainder else 0)    // first `remainder` people absorb the odd cent
        shares[userId] = Money(cents, total.currency)
    assert sum(shares.values()) == total              // exact, by construction
    return shares


GreedyDebtSimplifier.simplify(ledger) -> List<Transfer>
    // PURE. Reads the ledger, mutates nothing. The caller decides what to do with the result.
    creditors = ledger.creditors()      // [(user, +amount)], amount > 0
    debtors   = ledger.debtors()        // [(user, +amount)], magnitude of what they owe

    // sort descending by magnitude: greedily matching the largest pair first zeroes one side
    // per transfer, which is what bounds the result at (n - 1) transfers.
    creditors.sortDescendingByAmount()
    debtors.sortDescendingByAmount()

    transfers = []
    i = 0; j = 0
    while i < creditors.size() and j < debtors.size()
        c = creditors[i]; d = debtors[j]
        amount = Money.min(c.amount, d.amount)

        transfers.add(new Transfer(from = d.user, to = c.user, amount = amount))

        c.amount = c.amount.minus(amount)
        d.amount = d.amount.minus(amount)
        if c.amount.isZero() : i++          // exact — integer minor units, so == 0 is safe
        if d.amount.isZero() : j++

    return transfers            // at most n-1 transfers for n users with non-zero balance


ExpenseGroup.recordSettlement(from, to, amount) -> SettlementId
    if from not in members or to not in members  throw NotAMember
    if from == to                                throw InvalidSettlement("cannot pay yourself")
    if not amount.isPositive()                   throw InvalidSettlement("amount must be positive")
    if amount.currency != this.currency          throw InvalidSettlement("currency mismatch")

    // Partial settlement needs NO new state and NO new enum value: paying less than you owe
    // simply moves your net balance closer to zero. "PARTIALLY_SETTLED" is a state that only
    // has to exist if you chose to store dues as mutable rows.
    owed = ledger.netOf(from).negate()            // positive if `from` owes money overall
    if amount > owed
        throw InvalidSettlement("cannot settle more than the outstanding balance")

    s = new Settlement(newId(), from, to, amount, now())
    settlements.add(s)
    ledger.apply(s.netEffect())
    return s.id


ExpenseGroup.outstandingTransfers() -> List<Transfer>
    return simplifier.simplify(ledger)      // derived on demand; nothing accumulates, ever
```

**Edge cases covered:** zero/negative totals · currency mismatch · non-member payer or participant ·
payer contributions not summing to total · `$10 / 3` remainder (deterministic, sums exactly) ·
percentages not summing to 100 (rejected in `PercentageSplit`) · self-settlement · over-settlement ·
a user who both paid and consumed (nets out automatically in `netEffect`) · removing a member with a
non-zero balance · empty participant list.

**The trace that finds bugs** — not two clean expenses, but *simplify → new expense → simplify again*:

| Step | ledger.net |
|---|---|
| A pays $90, split 3 ways | A +60, B −30, C −30 |
| B pays $60, split 3 ways | A +40, B +10, C −50 |
| `outstandingTransfers()` | C→A $40, C→B $10 *(derived; ledger untouched)* |
| C pays A $25 (partial) | A +15, B +10, C −25 |
| `outstandingTransfers()` | C→A $15, C→B $10 — **correct, with no state flags at all** |

## 5. Design decisions, each against a named alternative

| Decision | Alternative rejected | What it gives up |
|---|---|---|
| **Derived balances** — a `BalanceLedger` of net positions; dues computed on demand | **Stored `Due` rows with lifecycle states** (his design) | Loses "this specific due came from the dinner" as a first-class row. Buys: no SIMPLIFIED/SETTLED bookkeeping, no unbounded growth, no double-count risk, no strategy-mutates-input problem. Per-expense attribution is still available by replaying `expenses` — it just isn't the primary structure. |
| **`Money` with integer minor units** | `double` amounts | Slightly noisier arithmetic. Buys: exact `== 0` comparisons in the simplifier loop, and remainders that provably sum to the total. **His `if creditor.amount == 0` is only safe under this choice.** |
| **`SplitStrategy` interface** | `switch (splitType)` inside `Expense` | One more type per split kind. Buys: new split types are additive — the one place a pattern genuinely pays, because the requirement literally says "decide how the expense should be shared". |
| **`DebtSimplifier` as a pure function** | A strategy that restamps the dues it read (his design) | Nothing. A method named `simplify` that also performs state transitions is doing two jobs; making it pure costs zero and removes a class of ordering bugs. |
| **`ExpenseGroup` as aggregate root; the service is a thin facade** | Manager/service classes holding money rules | Group becomes the biggest class. Buys: every money invariant has exactly one home — the object that owns `members` and `currency`, which are precisely what the rules need. |
| **`recordSettlement(from, to, amount)` — ids and values, never objects** | `settleDue(due: Due)` (his design) | An extra lookup. Buys: a forged or stale object is structurally impossible. |
| **No Factory, no Singleton, no Builder** | — | Nothing. Constructors and a repository suffice. **He got this right and it is worth keeping.** |

## 6. Extensibility — where each follow-up lands

| Follow-up | Seam | Change |
|---|---|---|
| **Multiple payers** | `Expense.paidBy: Map` | **Zero change** — modelled from the start; `netEffect()` already credits each payer their own contribution. *(His design needed a field change plus a rewrite of due generation.)* |
| **Partial settlement** | `Settlement` + ledger | **Zero change** — paying less than you owe just moves your net closer to zero. No `PARTIALLY_SETTLED` state exists because there are no due rows to put a state on. |
| **New split type (shares, itemised)** | `SplitStrategy` | One new class. Nothing else moves. |
| **Multi-currency** | `Money` | Add FX at the group boundary; `Money` already carries currency and rejects mismatches. |
| **"What did the dinner cost me?"** | `expenses` list | `expense.netEffect()[me]` — per-expense attribution survives because expenses are immutable facts. |
| **Recurring expenses** | `ExpenseAppService` | A scheduler calling `addExpense`. No model change — the group can't tell the difference. |

## 7. Concurrency

**Category: correctness** — `addExpense`, `recordSettlement` and `outstandingTransfers` all perform
read-modify-write (or read-a-consistent-snapshot) against `BalanceLedger`. Interleaving a settlement with
a simplification produces transfers computed from a ledger that changed underneath the reader.

**Primitive: one `ReentrantReadWriteLock` per `ExpenseGroup`, guarding `ledger`.**
- Write lock: `addExpense`, `recordSettlement`, `addMember`, `removeMember`
- Read lock: `outstandingTransfers`, `balanceOf`

**Where it lives:** inside `ExpenseGroup` — not in the service, not in the simplifier. The lock guards the
invariant, so it belongs on the object that owns the invariant. `GreedyDebtSimplifier` stays pure and
lock-free: it receives a consistent snapshot and knows nothing about threading. Callers can never forget
to lock, because they never touch `ledger` directly.

**Cost:** all writes to one group serialise. Acceptable — contention is *per group*, and a group is
~10 people, so the lock is essentially uncontended in practice while different groups proceed fully in
parallel. Reads don't block each other. Deadlock risk is nil: no operation acquires two group locks, and
lock scope never spans a call into user code. If cross-group operations were ever added they would need a
global lock ordering (e.g. by group id) — worth saying out loud, and worth not building until needed.

**Why not coarser:** a single global lock serialises unrelated groups. **Why not finer** (per-user atomics
on each balance): a transfer touches two balances and must be atomic across both, so per-balance atomics
would reintroduce exactly the race they were meant to remove.
