# LLD Round Transcript
**Date:** 2026-08-08
**Start Time:** 16:18 · **End Time:** 17:06 · **Duration:** 48 min
**Problem:** Expense Splitter (Splitwise)
**Category:** product domain
**Performance Rating:** 3/5  <!-- machine-read on future rounds; <=3 = eligible for re-ask, >=4 retired -->
**Hints Used:** 0/2
**Requirements Asked:** whether the app is single-threaded / in-memory or multi-threaded · **Never Asked:** multiple currencies · multiple payers per expense · partial settlement of a due · what happens to dues when a member leaves · whether split types (equal/exact/percentage/shares) need to be first-class · what simplifyDebts does with already-settled dues
**Out of Scope list produced:** Unprompted (payment integration, notifications)
**Self-Verified:** No — never traced until asked; when asked, his trace was correct (verified independently)
**Concurrency follow-up:** correctness (T1 settleDue interleaved with T2 simplifyDebts) — named the category before reaching for the lock; named a lock; stated the cost only when asked; never named where the lock lives
**Would it have fit a real 40-min round?** Yes, just barely — he'd have been mid-concurrency-answer at the buzzer.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements + Out of Scope | 5 min | 3 min | -2 | Yes |
| Entities & relationships | 8 min | 20 min | +12 | No |
| Class design | 20 min | 24 min | +4 | Marginal |
| Implementation + trace | 32 min | 39 min | +7 | Marginal |
| Extensibility + concurrency | 40 min | 48 min | +8 | Marginal |
| **Total** | 40 min | **48 min** | +8 | Marginal |

---

## Problem Statement

> Design an expense-sharing application — a Splitwise.
>
> That's the whole prompt. Everything else is yours to establish. Single process, in-memory. No QPS, no
> databases, no sharding. The deliverable is classes, state, method signatures, and working core logic.

---

## Conversation Log

**Interviewer:** Design an expense-sharing application — a Splitwise. Five phases: requirements + out of scope (5 min), entities (8), class design (20), implementation + trace (32), extensibility (40). Clock measured, not enforced. No UML. Which language for phase 4? Any questions before you start?

**Aayush:**
Here are what i think the requirements should be ->
1. User can create an expense group and add their friends to it.
2. User can add expenses to the group and decide how the expense should be shared and among whom it should be shared.
3. Users should be able to simplify the debts to finally get what amount they owe and to whom.
4. Users should be able to mark their dues as settles in the app.

Out of scope ->
1. Payment integration
2. Notifications

Are there any requirements i am missing or any exceptions i need to beware of

**Interviewer:** Requirements and an explicit Out of Scope list, unprompted — good. On "what am I missing?" — that's yours to decide; the scope is your call and I'll hold you to whatever you set. If you have a specific question about the domain, ask it and I'll answer precisely and commit. You also haven't told me your language.

**Aayush:** do i need to have multi threaded cocnerns or will this be a single threaded in memory app ?

**Interviewer:** Design it single-threaded, in-memory. Don't build synchronization into the core model. I may hand you a concurrency scenario as a follow-up later.

**Aayush:**
pseudocode
Entities & Classes ->
User (id, name, email)
ExpenseGroup (id, simplifyDebtsOn, members: User[], expenses:[], debtSimplificationStrategy)
Expense (postedBy: User , amount , createdAt, , sharingDict:{userId : share %})
UserManager (users: User[])
ExpenseGroupManager(expenseGroups: ExpenseGroup[])

**Interviewer:** Four probes. (1) `sharingDict: {userId: share%}` — which requirement makes percentage the representation? Someone pays a $100 dinner and says "Aayush owes exactly $23.50, split the rest evenly." Can your Expense express that as written? (2) `simplifyDebtsOn` and `debtSimplificationStrategy` are two fields for one concern — which requirement needs each? (3) FR4 — I owe you $40, pay cash, mark it settled. Which object changes and what does it record? (4) Which of these orchestrates?

**Aayush:**
1. sharingDict should instead represnet the actual amount each person needs to pay and the client can actually convert the percentages to actual amount and save them in BE
2. simple debtSimplicationStragey woudl work
3. The expenseGroup would also have another entity called Dues where each due would be details of who owes whom how much. These dues will be input to the debtimplificationStrategy which will return simplfied dues for the expense group.
4. give me a smal himt

**Interviewer:** No hint — that one's yours. You have UserManager and ExpenseGroupManager; a caller who wants to add an expense calls *something*. Tell me what, and what it returns. And #3 is only half answered: I hand you cash and tap "settle" — what changes? Does a Due get deleted, does its amount go to zero, does something get appended? And what happens to it the next time an expense is added?

**Aayush:** dues state gets altered to settles Dues will have a state. the caller will call the add expense method of the expsne group , but to do this it must have the epxense grou which would be present in the expsne group maanger . SO there needs to be some centralised class which holds the user maanger and expnse group maanger and exposes mthods to get the epxnse group and create new users etc

**Interviewer:** Good — you got to the central entry point yourself. Phase 3: class design. State as `- field: Type`, public API as `+ method(args): ReturnType`, for every class including the central one and Due.

**Aayush:** let the central class be ExpenseAppOrchestrator

**Aayush:** *(full class outlines — see design sheet: User, Expense, Due, ExpenseGroup, DebtSimplificationStrategy interface, DefaultDebtSimplificationStrategy, UserManager, ExpenseGroupManager, ExpenseAppOrchestrator, plus a typical-flow snippet)*

**Interviewer:** Four probes. (1) `validateExpense` — enumerate exactly what it checks, specifically the relationship between amount, sharingDict and members. (2) `simplifyDebts()` does `dues = simplifiedDues`. Group has three dues, I settle one, then simplifyDebts() runs. What's in dues afterwards, what happened to the settled one, and can anyone still answer "what did the dinner make me owe"? (3) `settleDue(due)` takes a caller-supplied Due — what stops me constructing `new Due(you, me, 5000)`? How, concretely? (4) "create dues from expense" — which object *knows* how to do that, and why is that the right home?

**Aayush:**
1. it must validate that tge breakdown sums upto whole and the amount shared between group memebers
2. The simpifyDebts should add new simplified debts and mark old debts status as simplfied
3. The group must maintain a list of dues.
4. Expense object should be responsible for generating the dues necessray since it has access to all the information of shring dict amoint memebers

**Interviewer:** On #3 you named where dues live but not the check itself. What line of code in settleDue decides this Due is legitimately one of the group's? Then phase 4: write real pseudo-code for `Expense.generateDues()` including validation, and `DefaultDebtSimplificationStrategy.simplify(dues)` — that's FR3, the hardest logic here. Include your edge cases.

**Aayush:** *(submits generateDues — validates amount > 0, participants are group members, no negative shares, breakdown sums to total; then builds dues skipping zero shares and the payer — see design sheet)*

**Interviewer:** (1) Give me the full signature of generateDues — what does it take, what does it return? (2) Now the simplify algorithm.

**Aayush:** generateDues does not take in anything it is called on the expnse object and it returns a list of dues. find all the creditors i.e people who are owed money and the debtors i.e people who are supposed to pay, by iterating through all the active dues... after simplification it does not really matter whether the original debtor is still paying the same guy, what matters is that all creditors get their money and debtors pay their debts. So simply sort creditors and debtors and start assigning each creditor until they are satisfied, with a debtor until they still need to pay. The active Dues will then be marked as simplified state so that they dont affect future calculations

**Interviewer:** Write that as pseudo-code — actual loops and state, same detail as generateDues. Prose describes the idea; the code is where the off-by-ones live.

**Aayush:** *(submits simplify in two parts — active filter, net balance map, creditor/debtor lists, greedy two-pointer min-matching, then restamps the originals SIMPLIFIED and returns the new list — see design sheet)*

**Interviewer:** Now walk one concrete scenario through your code. Members A, B, C. (1) A pays $90 dinner {A:30,B:30,C:30}. (2) B pays $60 taxi {A:20,B:20,C:20}. (3) simplifyDebts(). (4) C settles. What does group.dues contain after each step, with states?

**Aayush:** now with this design the expenseGroup will keep accumulaitng dues which have been simplified as well as settled

**Interviewer:** That's an observation about growth, not a trace. Walk the four steps.

**Aayush:**
1. Dues -> B->A, 30 not settled, C->A 30 not settled
2. Dues -> B->A 30 X , C->A 30 X , A->B 20 X, C->B 20 X
3. Dues -> B->A 30 simplified, C->A 30 simp, A->B 20 sim, C->B 20 simp, C->A 40 X, C->B 10 X
4. same but C debts are settled state now

**Interviewer:** Trace recorded, taken at face value. Phase 5, verbal only. Follow-up 1 — multiple payers: $200 hotel, A paid $150, B paid $50, split three ways. Your Expense has a single postedBy and generateDues hard-codes creditor = postedBy. Seam or rewrite, and which class changes? Follow-up 2 — partial settlement: C owes A $40 and pays back $25. Due.settle() is all-or-nothing. What changes, and where does "a due may be paid down incrementally" live?

**Aayush:**
1. Multiple payers - Expense class gets updated postedBy -> paidBy:{ UserId: amount}. Generate dues method will now need to generate dues considering the contribution of multiple payers , so the debtors net payment will be split on the percentage of payment made by the user who actually paid. So the payment split of multiple payers is considered for determining the dues.
2. The due class will own the logic to paritally settle a due it will record each settlement effort. And now have an extra tate called PARTIALLY SETTLED

**Interviewer:** Follow-up 3 — concurrency. T1: C calls group.settleDue(cOwesA40). T2: A calls group.simplifyDebts(). They interleave. (1) What concretely breaks — walk the interleaving. (2) What category. (3) Smallest primitive, where in your classes it lives, and what it costs.

**Aayush:**
1. This is correctness problem if t1 marks debt as settled while t2 is still processing then the actual simplfied debt which would be added by the simplifcation woudl be wrong as they would not account for the simplfied debt.
2. This is correctness problem
3. It will be fixed by using a lock before simplfiyinf debts to allow no other thread to push new dues or modify exisitnf dues of the expsen group.

**Interviewer:** You named the category and the primitive. You haven't said what it costs.

**Aayush:** It costs lower throughput for adding dues but this is fine for a small scale app. Reduced throughput for settling and adding new dues in exchange for correctness of debt simpifcation

**Interviewer:** Round ends.

---

## His Design

**Requirements he gathered:**
1. User can create an expense group and add friends to it.
2. User can add expenses to the group and decide how and among whom the expense is shared.
3. Users can simplify debts to get what they owe and to whom.
4. Users can mark dues as settled.

**Out of Scope:** payment integration, notifications.

**Entities & relationships:** User · Expense · Due (owned by ExpenseGroup, has lifecycle state) · ExpenseGroup (members, expenses, dues, debtSimplificationStrategy) · DebtSimplificationStrategy interface + DefaultDebtSimplificationStrategy · UserManager · ExpenseGroupManager · ExpenseAppOrchestrator as the facade holding both managers.

**Class outlines / Core implementation:** see `expense_splitter_design.md` (full outlines, `generateDues`, `simplify`, the four-step trace, and all three follow-ups).

**Gaps / misplaced responsibilities:**
- `generateDues()` declared to take no parameters, but its body references `groupMembers` and `findUser(userId)` — neither of which exists on `Expense`. Confirmed under direct questioning and never caught. Membership validation cannot run and the debtor cannot be resolved as written.
- `simplify()` mutates its input (restamps originals SIMPLIFIED) while its contract is `simplify(dues) -> List<Due>` — two responsibilities in one method, and it lives in the strategy rather than in the owner of `dues`.
- `ExpenseGroup.simplifyDebts()` still reads `dues = simplifiedDues` in the written outline, which discards the history the strategy's own comment claims to preserve. Corrected verbally, never in code.
- `settleDue(due: Due)` takes a caller-constructed object; the concrete membership check was asked for twice and never given.
- `User` is a pure getter bag; `getDues()`, `getShares()`, `getMembers()` exist mainly so callers can make decisions.
- No currency/Money type; `amount` is a bare number, and float equality (`== 0`) is used as a loop-advance condition.
- Unbounded growth of `dues` (SIMPLIFIED + SETTLED accumulate forever) — he noticed this himself but never addressed it.

---

## Feedback Given

### Round conditions
Hints used **0/2** — he asked for one on the orchestrator question; it was declined and he got there himself. No ceiling applies. Requirements asked unprompted: the threading question only. Out of Scope list: **unprompted**. Self-verified: **no** — never traced until asked, but the trace he then produced was **correct** (verified independently). Concurrency: category named **before** the lock; cost stated only when asked.

### Rubric

**Requirements & scoping — 4/5.** FRs and an explicit Out of Scope list, unprompted, before writing a single class — neither of which he produced last round. Also asked the threading question unprompted. Held off 5 by asking only one real clarifying question, and by "are there any requirements I'm missing?", which asks the interviewer to do the scoping.

**Entity modelling — 4/5.** Right objects at the right granularity, no God class and no micro-object soup. `Due` as a first-class entity with its own lifecycle state is correct and he reached it under one probe.

**Class design — 3/5.** Complete-ish signatures, but `User` is a pure getter bag and `Expense` is nearly one.

**Responsibility placement — 4/5.** Genuine improvement. Due-generation → Expense, justified by what state it holds. Partial settlement → Due, unprompted. `Due.settle()` guards its own double-settle. Still Ask-shaped at the boundary: `settleDue` trusts a caller-supplied object and the getters exist so callers can decide.

**Implementation & correctness — 3/5.** The greedy net-balance simplification is the right algorithm and the code is correct. Validation is genuinely thorough — non-positive amount, non-member participant, negative share, sum mismatch, zero-share skip, payer skip. But `generateDues()` references `groupMembers` and `findUser` which `Expense` does not have, asserted as parameterless under direct questioning. And `simplify()` mutates its input while `simplifyDebts()` replaces the list, so the written code discards the history the comment claims to keep.

**Simplicity & judgement — 5/5.** The standout. Exactly one pattern — Strategy — justified by an actual requirement, with a named interface and a default implementation. No Factory, Singleton or Builder theatre. When `simplifyDebtsOn` was questioned he deleted it rather than defending it.

**Extensibility — 5/5.** Both follow-ups landed on a seam with the right class named first attempt. Multiple payers → `paidBy: {userId: amount}` with proportional attribution, and he saw the creditor side becomes a distribution. Partial settlement → Due owns it, records each attempt, adds PARTIALLY_SETTLED. No rewrites proposed.

**Concurrency — 3/5.** Category named before the primitive. Interleaving explanation directionally right. But the lock was described as a *phase* ("before simplifying debts") rather than a *place* — never said it lives on ExpenseGroup guarding `dues` — and the cost came only after two asks.

**Communication — 4/5.** Concise, responsive, revised rather than defended four separate times. Still under-answers: "the group must maintain a list of dues" was never turned into the concrete check asked for twice.

### Pace report
Requirements 3 min (ref 5), entities 20 (8), class design 24 (20), implementation + trace 39 (32), extensibility + concurrency 48 (40). Total 48 vs 40.

**Would this have fit a real 40-minute round? Yes, just barely** — mid-concurrency-answer at the buzzer, but everything that matters lands. A sharp improvement over the same day's system design round.

**Biggest time sink: entities, 12 minutes over.** Not slowness — entities took four exchanges (post, probe, revise, probe, revise). Requirements took 3 minutes; entities took 17. One more requirement question up front ("can an expense have multiple payers? can a due be partially paid?") removes most of that probing — and both of those were exactly the extensibility follow-ups. The questions he didn't ask in phase 1 returned as the follow-ups in phase 5.

### Performance Rating: 3/5 — Pass
No ceiling binds — zero hints, Out of Scope unprompted, code reached. An honest 3 at the top of the band: a workable design that needed real prompting on responsibilities and edge cases. It was a 4 on model quality, judgement and extensibility; what holds it at 3 is the five things that only appeared after pushing (validateExpense contents, simplify/history semantics, settled-due interaction, the trace, the lock's cost) plus a core method referencing state its class doesn't have, asserted as correct when challenged.

### Senior-signal scorecard
- **Scopes before designing — Strong** (was Mixed). FRs + Out of Scope + a threading question, all unprompted, before any class.
- **State derived from requirements — Mixed.** Deleted `simplifyDebtsOn` under challenge rather than defending it. But `sharingDict` as percentages was speculative and had to be revised, and Expense lacks the state its own method depends on.
- **Rules live with their state — Mixed** (was Weak). Due-generation → Expense, partial settlement → Due, double-settle guard inside Due. Still Ask-shaped at the boundary.
- **Simplicity held under pressure — Strong.** One justified pattern, zero ceremony, a field deleted when it couldn't be justified.
- **Verifies own logic — Mixed** (was Weak). Never volunteered a trace, but the trace was correct and he spontaneously noticed unbounded due growth.
- **Extends without rewriting — Strong** (was Mixed). Both follow-ups on the right class, first attempt.

**Overall: solidly senior on modelling and judgement, mid-level on verification. Hire.** Not strong-hire only because a senior would have caught the `generateDues` collaborator gap the moment the signature was asked for — that question was the opening and he walked past it.

### What a senior strong-hire would have done on this problem
- **Asked three more requirement questions, all of which would have paid.** "Can an expense have more than one payer?" is follow-up 1 — asking it in minute 3 means `paidBy: Map` from the start instead of a revision in minute 44. "Can a due be paid partially?" is follow-up 2. "Multiple currencies?" is the one nobody asked, and would have made `amount` a `Money` type instead of a bare number.
- **Given Expense its collaborator, or moved the validation.** `generateDues(members: List<User>): List<Due>` — Expense owns the rule but not the roster, so the group passes it in. Or keep it parameterless and move membership validation to `ExpenseGroup.validateExpense`, where the roster lives. Either is right; asserting the method needs nothing is not.
- **Made settleDue take an id, not an object.** `settleDue(dueId: String)` — the group looks it up in its own dues, and a forged Due becomes unconstructable. "The group must maintain a list of dues" was the right instinct; `settleDue(due: Due)` throws it away. Same finding as last round's "trusts objects passed in by callers", in a different costume.
- **Not mutated the strategy's input.** `simplify()` returning a new list *and* restamping the old ones is two responsibilities. The state transition belongs in `ExpenseGroup.simplifyDebts()`, which owns dues; the strategy should be a pure function of balances. That also fixes the `dues = simplifiedDues` contradiction, because the group would append rather than replace.
- **Traced without being asked** — and picked the trace that hurts: simplify, then add a new expense, then simplify again. That's where SIMPLIFIED/ACTIVE interaction either holds or falls apart.
- **Placed the lock, not the phase.** "A ReentrantLock on ExpenseGroup guarding dues, held by addExpense, settleDue and simplifyDebts. Cost: every group operation serializes — fine, contention is per-group and a group is ~10 people. Deadlock risk nil, no operation takes two group locks."

### One drill
Take today's design and write `generateDues` and `simplifyDebts` as real, compiling C# or Python — not pseudo-code. Nothing exposes a missing collaborator faster than a compiler. The gap wasn't judgement; it was that pseudo-code let `findUser(userId)` look like a decision when it was a hole. Then re-read the checklist in `lld_senior_guidance.md`, specifically signal 5 and the "no getter exists purely so a caller can make a decision" line.
