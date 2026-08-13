# LLD (Low-Level Design) — the senior bar

Read this **between** rounds, not during one. It is the standard `/lld-round` grades against.

LLD is not system design. System design is the map — traffic, storage, sharding, consistency. LLD is the blueprint for one building on that map: classes, state, method signatures, and how objects collaborate for a *single self-contained* feature. You will be asked for real code or tight pseudo-code, not boxes and arrows.

---

## The five phases (~35–45 min)

| Phase | Reference | What you must produce |
|---|---|---|
| 1. Requirements | ~5 min | Numbered functional requirements **plus an explicit Out of Scope list**, confirmed with the interviewer |
| 2. Entities & relationships | ~3 min | Entity list + who owns/uses whom + which one orchestrates. Simple boxes and arrows, never formal UML |
| 3. Class design | ~12 min | Per class: state variables with types, public methods with signatures and return types |
| 4. Implementation | ~10 min | Code or pseudo-code for the core methods, edge cases, then a **trace of one concrete scenario** |
| 5. Extensibility | ~5 min | Verbal-only answers to "what if we also need X" — point at the seam, don't rewrite |

Ask the interviewer which they want in phase 4 — pseudo-code or a real language — before you start typing. Use the language you actually know.

---

## Phase 1 — the requirements walk (8 items)

The LLD prompt is deliberately under-specified: a domain and nothing else. The failure mode is improvising a different set of questions every round and missing the ones that would have changed the design. This is the fixed walk — run all eight, in order, out loud, and write the answers down. It takes about four minutes.

| # | Item | What you actually ask | Why it changes the design |
|---|---|---|---|
| 1 | **Actors & entry point** | Who calls this — one client, many, a UI, another service? What's the single object they talk to? | Names the orchestrator before you design a class. |
| 2 | **Core operations** | The verbs this thing must support, listed. "Park, unpark, price" / "publish, subscribe, ack". | This *is* the public API. Everything else exists to serve it. |
| 3 | **Rules & legality** | For each verb: what makes a call legal or illegal? Capacity, ordering, ownership, uniqueness. | Rules are where state comes from — no rule, no field. |
| 4 | **Lifecycle & terminal states** | Does an entity move through states? What ends it? Can it be undone/cancelled/reopened? | Decides whether you need a state machine or just a flag. |
| 5 | **Failure behaviour** | On an illegal call: throw, return false, return an Optional/Result, or silently no-op? **Pick one and hold it for every method.** | Half the "incomplete signature" findings are really an unmade decision here. |
| 6 | **Multiplicity & domain variants** | How many of each thing? Sizes/types/tiers? Currencies? Multi-payer, multi-owner, partial? | This is the question that turns a toy design into a real one, and the one you skip. |
| 7 | **Concurrency posture** | Is this object shared across threads, or single-threaded? Assume single unless told otherwise — **but ask, don't assume silently.** | Decides now whether state must be guarded; retrofitting locks late is a rewrite. |
| 8 | **Explicit Out of Scope** | State it back: persistence, auth, networking, UI, pricing, whatever you're excluding. Get agreement. | Never optional. Its absence caps the round at 3/5 on its own. |

Three rules on top of the walk:

- **Every rule you raise gets a resolution.** If you say "what if the locker is too small for the package?" you do not move on until there's an answer written down. A raised-and-abandoned rule is worse than never raising it — it shows you saw the edge and dropped it.
- **Write it as a numbered list**, not prose. FR1…FRn, then Out of Scope. You will reference the numbers when justifying state later ("R3 is why Board tracks column heights").
- **Assume-and-state beats silence.** If the interviewer won't answer, say "I'll assume single-threaded, single currency, no persistence" out loud and design against it. An unstated assumption is scored as a miss; a stated one is scored as scoping.

---

## The six senior signals

1. **Scopes before designing.** Asks about primary capabilities, rules and completion conditions, error handling, and boundaries — then *states what is out of scope* and gets agreement. Assuming instead of asking is the most common junior tell.
2. **State is derived from requirements, not invented.** Every field traces back to a requirement ("to enforce R3 the Board must track column heights"). No speculative fields, no bag-of-getters data classes.
3. **Rules live with the state they act on — "Tell, Don't Ask."** Objects expose behavior, not getters that let callers make the decisions. Workflow and lifecycle rules belong in the orchestrator; data rules belong in the entity that owns the data. Reaching through objects (`order.getCustomer().getAddress().getZip()`) is the anti-signal.
4. **Simplicity held under pressure.** KISS and YAGNI are the most-violated principles in this round. A Factory, Builder, or Singleton with no justification reads as overengineering, not sophistication. Most strong designs use **zero or one** pattern. Add complexity when simplicity stops working, and say why at that moment.
5. **Verifies own logic.** Enumerates edge cases (invalid input, illegal operation, out-of-range, state violation) and then traces a non-trivial scenario step by step, naming state after each step. This is what catches "forgot to switch turns" and "win never triggers".
6. **Extends without rewriting.** Under follow-ups, points at the existing seam (an interface, a strategy, the orchestrator boundary) rather than bolting on a special case or redesigning. Interviewers reward designs that *adapt*, not designs that anticipated everything.

At senior level expect **multiple sequential** follow-ups, not one.

---

## Patterns — the short, honest list

- **Strategy** — the single most common pattern in LLD. Tell: an if/else or switch on a type, or "different approaches to the same task".
- **Observer** — tell: "notify", "update multiple components when X changes".
- **State machine** — tell: the word *state* recurring in the requirements, complex legal transitions. When it fits it becomes the centrepiece of the design; draw the transition diagram.
- **Facade** — you build one every time you make an orchestrator. Naming it matters less than the clean entry point.
- **Factory / Builder / Singleton / Decorator** — situational and easy to overuse. Singleton in particular: prefer passing the dependency through the constructor.

US interviews reward the reasoning; they rarely ask for the pattern's name. Naming a pattern you can't justify costs more than staying silent.

## SOLID

Know it, don't perform it. Excessive SOLID is falling out of fashion outside Java/C#. SRP and DIP earn their keep in a 40-minute round; ISP and LSP mostly surface as anti-patterns to avoid (a subclass that throws `NotImplemented`, an interface forcing unused methods). Never break KISS to satisfy SOLID.

---

## Concurrency — expected at senior level

It arrives either as a follow-up to a plain problem ("now two cars race for the last spot") or as the problem itself (thread pool, rate limiter, connection pool, scheduler). Three categories, and naming the category is half the answer:

| Category | What breaks | Primitives |
|---|---|---|
| **Correctness** | Shared state corrupted by concurrent updates — check-then-act, read-modify-write | locks, atomics/CAS, thread confinement, immutability |
| **Coordination** | Threads need handoff or ordering — producer/consumer, async processing | blocking queues, condition variables, event loop, actors |
| **Scarcity** | Limited resources, unlimited demand — 10 connections, 100 requests | semaphores, resource pools, bounded queues |

Say which category the problem is, then name the smallest primitive that fixes it, then say what it costs (contention, lock scope, fairness, deadlock risk).

---

## Pre-round checklist

- [ ] Ran all eight items of the requirements walk, including multiplicity/variants and concurrency posture.
- [ ] Wrote the requirements down as a numbered list, and an explicit Out of Scope list.
- [ ] Every rule raised got a resolution — nothing left hanging.
- [ ] Committed to one failure convention (throw / false / Result) and used it in every signature.
- [ ] Named the orchestrator before designing any class.
- [ ] Every field justified by a requirement.
- [ ] No getter exists purely so a caller can make a decision the object should make.
- [ ] Used zero or one pattern, and can justify the one.
- [ ] Listed edge cases before being asked.
- [ ] Traced one concrete scenario out loud, naming state changes.
- [ ] Answered the extensibility follow-up by pointing at a seam, not by rewriting.
- [ ] For senior: named the concurrency category before reaching for a lock.
