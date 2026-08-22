# LLD Round Transcript
**Date:** 2026-08-22
**Start Time:** 10:23:34 · **End Time:** 11:10:04 · **Duration:** 47 min
**Problem:** Rate Limiter
**Category:** Infrastructure object (concurrency-first)
**Difficulty:** 3/5 — one rule to place, one genuine trade-off (where per-client state lives), and a concurrency curveball that the design must absorb without a rewrite
**Performance Rating:** 2/5  <!-- machine-read on future rounds; ≤2 = eligible for re-ask, ≥3 retired -->
**Hints Used:** 0/2
**Requirements Asked:** core operations · legality rule (rolling window) · caller identification and system inputs · what the caller sees when rate limited · per-client vs global limits (asked at +8 min, after phase 1 closed) · **Never Asked:** lifecycle/expiry of client entries · concurrency posture · variants beyond per-client (tiers, multiple simultaneous limits, number of distinct clients) · behaviour on unknown/unconfigured client
**Walk coverage:** 4/8 hit (+2 partial, 2 miss)
**Out of Scope list produced:** Unprompted (one item — distributed rate limiting)
**Self-Verified:** No — trace only on request, and his claimed trace contradicted his own eviction predicate at t=11
**Concurrency follow-up:** Correctness / check-then-act race on shared per-client state — handled excellently: named the category before the primitive, proposed a coarse lock, stated its throughput cost, refined to per-clientId lock striping
**Would it have fit a real 40-min round?** No — cut off during the second extensibility follow-up, losing the check-then-commit insight and the entire concurrency exchange

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements + Out of Scope | 5 min | 5 min | 0 | ✅ on pace |
| Entities & relationships | 8 min | 16 min | +8 min | ❌ over |
| Class design | 20 min | 24 min | +4 min | ❌ over |
| Implementation + trace | 32 min | 30 min | −2 min | ✅ ahead |
| Extensibility + concurrency | 40 min | 47 min | +7 min | ❌ over |
| **Total** | 40 min | 47 min | +7 min | ❌ over |

---

## Problem Statement

**Design a rate limiter.**

**Difficulty: 3/5**

Five phases, reference timeline (~40 min), measured but not enforced: requirements + out of scope by 5 min · entities & relationships by 8 · class design by 20 · implementation + trace by 32 · extensibility + follow-ups by 40. No UML. Phase 4 in the language of his choice.

Requirements were withheld entirely — domain only, no entity list, no rules, no scale, no concurrency hint.

---

## Conversation Log

**Interviewer [10:23:34 · +0m]:** Presented the problem, the five phases, the reference timeline, the design-sheet path. Asked which language for phase 4 and whether he had questions.

**Aayush [+1m]:** what are the operations expected from the system ?

**Interviewer [10:24:35 · +1m]:** Two verbs: (1) check a request — given an incoming request, decide whether it is allowed through or rejected; (2) configure — a caller must be able to set up what the limit actually is before requests start flowing. That's the whole public surface I need. Still need your answer on phase 4: pseudo-code, C#, Python, or Java?

**Aayush [+1m]:** what are the rules for rate limiting ?

**Interviewer [10:25:40 · +1m]:** A client may make at most N requests in any rolling window of duration T. *Any* window, not fixed calendar buckets. If a request would make it N+1 within the preceding T, it is not allowed through. N and T are what the caller configures. Language for phase 4?

**Aayush [+3m]:** pseudocode

**Interviewer [10:26:04 · +3m]:** Pseudo-code it is — I'll hold you to consistent signatures and types. Anything else before you start?

**Aayush [+4m]:** how would the caller of the request be identified from request body, what would the inputs of the system be ?

**Interviewer [10:27:17 · +4m]:** Assume identification happened upstream — middleware pulls out an API key, user id, IP, whatever — and hands you an opaque client identifier string. Parsing HTTP is not your system's job. So the input to your check operation is a client identifier. Whatever else you think it needs, you tell me.

**Aayush [+5m]:** what should the caller see in case they are rate limited ?

**Interviewer [10:28:11 · +5m]:** Being rate limited is a normal, expected outcome, not an error — so no exception. The caller gets a plain allow/deny decision back and acts on it. Whether that decision is just a boolean or carries more is a design call I'm leaving to you.

**Aayush [+5m]:** *(submitted requirements — see His Design below)*

**Interviewer [10:28:57 · +5m]:** Recorded. Phase 2 — entities and relationships. What objects do you see, who owns whom, and which one is the orchestrator?

**Aayush [+8m]:** the rules for rate limiting are set for per client level ?

**Interviewer [10:30:59 · +8m]:** Yes — each client's budget is tracked independently. Client A's traffic never consumes client B's allowance. The values themselves are one global pair: a single (N, T) configured once and applied to every client.

**Aayush [+12m]:** *(entity list — Request, RateLimitManager, RateLimitStrategy interface with TokenBucket / SlidingWindowLog, RateLimitConfiguration)*

**Interviewer [10:35:57 · +12m]:** Three questions: (1) Which requirement made you introduce `RateLimitStrategy` as an interface with two implementations — what does it buy over just writing the rule I gave you? (2) `RateLimitManager` holds one `RateLimitStrategy`. Client A's budget is independent of client B's — where does client A's request history actually live in this model? (3) What does `Request` do, other than carry a string?

**Aayush [+14m]:** 1. It allows for dynamically changing the underlying rate limit algorithm that is being used. 2. RateLimitManager should hold the state per client and this should be passed to the strategy class each time. 3. Nothing.

**Interviewer [10:37:46 · +14m]:** (1) "Dynamically changing the algorithm" — is that a requirement I gave you, or one you're adding? What breaks if there's only ever one algorithm? (2) Write me the field. `RateLimitManager` holds per-client state — what is its declared type? Give me the line as it would appear in the class. (3) If `Request` does nothing, does it earn a place in the model?

**Aayush [+16m]:** 1. It is for extensibility purposes. Not to satisfy any current requirement. 2. it is better for the strategy to own the state for rate limit for each client since different states need to be stored for different algorithms. 3. No need for it.

**Interviewer [10:40:05 · +16m]:** Recorded. Phase 3 — class design. For each class: state variables with types, and public methods with full signatures and return types.

**Aayush [+23m]:** *(class outlines — see His Design)*

**Interviewer [10:46:36 · +23m]:** Three questions: (1) You told me the strategy owns the state *because different algorithms need different state*. But `RateLimitStrategy` — the interface — declares `clientState: Map<string, List<TimeStamps>>`. Is that the interface, or is that `SlidingWindowLog`? (2) `RateLimitStrategy.getRateLimitStatus(clientId)` has to decide against N and T. Only `RateLimitManager` holds `rateLimitConfiguration`. Where does the strategy get N and T from? (3) A client calls `getRateLimitStatus("A")` twice in a row, back to back, with no other traffic. Same answer both times, or different?

**Aayush [+24m]:** 1. it is SlidingWindowLog. 2. Configuration passed as a param to method by Manager. 3. Different answer since each call would consume one quota.

**Interviewer [10:47:58 · +24m]:** Recorded. Phase 4 — implementation. Give me the pseudo-code for the core logic: `SlidingWindowLog.getRateLimitStatus` and whatever the manager does to get there.

**Aayush [+27m]:** *(pseudo-code — see His Design)*

**Interviewer [10:50:53 · +27m]:** Recorded. Walk me through a concrete scenario against that code. N = 2, T = 10 seconds. Client "A" calls in four times, at absolute times t = 0, 1, 5, 11. Give me the exact return value of each of the four calls, and the contents of `clientState["A"]` after each.

**Aayush [+30m]:**
- after t=0 → clientState[A] → {0}, return value → {Allowed, quota: 1}
- after t=1 → clientState[A] → {0,1} → Allowed, quota: 0
- after t=5 → clientState[A] → {0,1}, request rejected, not allowed, quota: 0
- after t=11 → clientState[A] → {1,11}, allowed, quota: 0

**Interviewer [10:53:37 · +30m]:** Noted, taking that as given. Phase 5 — follow-ups, verbal only. **First:** clients now have tiers. Free = 100 req/min, premium = 10,000 req/min, enterprise = 1,000,000 req/hour. Same check operation, same entry point. Where does that land in your design, and what changes?

**Aayush [+33m]:** We would need to add a Client object since now each client has state of string + their tier. We would also need to add a configuration for each tier now. So the configuration would need to have a tier attribute as well. For each client when calling the strategy method we need to pass relevant config.

**Interviewer [10:56:12 · +33m]:** **Second, escalating:** enterprise now needs two limits at once on the same client — no more than 1,000 requests per minute *and* no more than 20,000 per hour. Both must hold; a request is allowed only if it passes both. Same entry point, same return shape. What changes?

**Aayush [+43m]:** we would need to store state for each limit separately and then apply the strategy on both state and if both states allow then the request is allowed.

**Interviewer [11:07:00 · +43m]:** Follow that through: the per-minute limit runs first, allows, and pushes `now` into its list. The per-hour limit then runs and denies. The caller gets rejected. What's in the per-minute limit's state now, and is that correct?

**Aayush [+44m]:** no first the determine should be done then finally when both allow then go ahead and insert in both lists

**Interviewer [11:07:40 · +44m]:** **Last one — concurrency.** Drop back to the single-limit design. N = 2, T = 10. Client "A" currently has exactly one timestamp in its list, so one slot free. Two request threads for client "A" hit `getRateLimitStatus` at the same instant, on two cores. What breaks, and what do you do about it?

**Aayush [+47m]:** both the requests would be allowed and finally the list might grow to size 3 — a correctness issue. This is because of the concurrent access to the state. A solution for this is to serialise the access pattern; this can be done by introducing a lock when trying to get rate limit status for a request — this will reduce throughput. A better solution would be to take locks on the clientId so that different clientId requests can still be processed.

**Interviewer [11:10:04 · +47m]:** Round closed; feedback delivered.

---

## Walk coverage (phase 1)

| # | Item | Hit/Partial/Miss | Evidence |
|---|---|---|---|
| 1 | Actors & entry point | **Hit** | "how would the caller of the request be identified from request body, what would the inputs of the system be?" |
| 2 | Core operations | **Hit** | "what are the operations expected from the system?" — first question out |
| 3 | Rules & legality | **Hit** | "what are the rules for rate limiting?" |
| 4 | Lifecycle & terminal states | **Miss** | Never raised — no question about entry expiry, idle-client cleanup, or reconfiguration mid-flight |
| 5 | Failure behaviour | **Partial** | "what should the caller see in case they are rate limited?" — asked and answered, but never converted into a convention: `setRateLimitConfiguration` throws, `getRateLimitStatus` returns a dict, unknown-client case never raised |
| 6 | Multiplicity & domain variants | **Partial** | "the rules for rate limiting are set for per client level?" — asked at +8 min, *after* the requirements list was submitted. Tiers and multi-limit never raised; both had to be served as follow-ups |
| 7 | Concurrency posture | **Miss** | Never asked. On a rate limiter — the most shared object in a service |
| 8 | Explicit Out of Scope | **Hit** | "Out of scope -> 1. Distributed rate limiting" — unprompted |

**4 Hit · 2 Partial · 2 Miss**

**Dangling rules:** none. Everything he raised, he closed — a genuine improvement over prior rounds.

**Silent assumptions:**

| Assumption | Cost |
|---|---|
| The client map lives forever | `clientState: Map<string, List<TimeStamp>>` never evicts. Unbounded memory growth reachable by an attacker sending one request per spoofed identifier — the defining failure mode for a rate limiter |
| Single-threaded | Entire object model designed with zero synchronization; the lock only appeared when the curveball was served at +44 min, and never entered the written design |
| A list already exists for every clientId | Code does `remove timestamps from list` and `list.push(now)` with no initialization path. His own trace at t=0 silently supplied the empty list |
| One global (N, T) forever | Confirmed by the interviewer, but assumed rather than asked — and the tier follow-up is exactly the requirement that punishes it |

---

## His Design

**Requirements he gathered:**

1. Operations —
   a. Check a request: given an incoming request, decide whether it is allowed through or rejected.
   b. Configure: a caller must be able to set up what the limit actually is before requests start flowing.
2. Rules — A client may make at most N requests in any rolling window of duration T. Any window, not fixed calendar buckets. If a request would make it N+1 within the preceding T, it is not allowed through. N and T are what the caller configures.

**Out of Scope:**

1. Distributed rate limiting.

**Entities & relationships (first pass):**

1. `Request(ClientID: string)`
2. `RateLimitManager(RateLimitStrategy: RateLimitStrategy, config: RateLimitConfiguration)`
3. `interface RateLimitStrategy` → implementations `TokenBucket`, `SlidingWindowLog`
4. `RateLimitConfiguration(maxRequests: int, timeFrame: time)`

*Revised under questioning:* `Request` dropped ("does nothing", "no need for it"). Per-client state moved from the manager to the strategy ("different states need to be stored for different algorithms"). `RateLimitStrategy` interface retained for extensibility, acknowledged as satisfying no current requirement.

**Class outlines:**

```
class RateLimitManager:
    - rateLimitStrategy: RateLimitStrategy
    - rateLimitConfiguration: RateLimitConfiguration

    + setRateLimitConfiguration(N: int, T: Time) -> void   // throws InvalidRequestCount, InvalidTimeWindow
    + getRateLimitStatus(ClientID: string) -> {"RequestAllowed": bool, "QuotaLeft": int}


class RateLimitStrategy:
    - clientState: Map<string, List<TimeStamps>>

    + getRateLimitStatus(clientId: string) -> {"RequestAllowed": bool, "QuotaLeft": int}
```

*Clarified under questioning:* `clientState` belongs to `SlidingWindowLog`, not the interface. Configuration is passed as a parameter to the strategy method by the manager. `getRateLimitStatus` is mutating — two back-to-back calls give different answers, "since each call would consume one quota".

**Core implementation:**

```
class RateLimitManager:
    + getRateLimitStatus(ClientID: string) -> {"RequestAllowed": bool, "QuotaLeft": int}
        return strategy.getRateLimitForStatus(clientId, rateLimitConfiguration)


class SlidingWindowLog:

    getRateLimitStatus(clientId, rateLimitConfig) ->

        now = current time.
        For the clientId remove the request timestamps from list where timestamp <= now - rateLimitConfig.T

        if list size >= N:
            deny request no quota left.

        else:
            list.push(now)
```

**Gaps / misplaced responsibilities:**

1. **Trace contradicts the code.** At `t=11` with `T=10`, the predicate `timestamp <= now - T` evaluates to `timestamp <= 1`, evicting **both** `0` and `1`. Real result: `{11}`, quota 1. He claimed `{1, 11}`, quota 0. He traced the algorithm in his head, not the code on the page. Never caught.
2. **The method never returns its declared value.** Signature declares `{"RequestAllowed": bool, "QuotaLeft": int}`. The deny branch is prose ("deny request no quota left"); the allow branch does `list.push(now)` and falls off the end. `QuotaLeft` is never computed in the body.
3. **No initialization for an unseen client.** Both branches operate on a `list` that is never created.
4. **Interface declares an implementation's field.** `clientState: Map<string, List<TimeStamps>>` on `RateLimitStrategy` forces every implementation into a timestamp-list state shape — which makes the promised `TokenBucket` (needing `tokens: double`, `lastRefill`) impossible. The abstraction added for extensibility is the thing blocking extension.
5. **Mutating method named as an accessor.** `getRateLimitStatus` consumes quota; confirmed intentional. A caller reading the signature to build a dashboard silently burns the client's budget. The `peek` / `tryAcquire` split he needed here is exactly the split he independently discovered two phases later on the multi-limit follow-up.
6. **Name mismatch inside his own two lines.** Manager calls `strategy.getRateLimitForStatus(...)`; the defined method is `getRateLimitStatus`.
7. **Unbounded map.** No eviction, no idle reclamation, on the one object in a service that faces attacker-controlled key cardinality.
8. **No synchronization anywhere in the written design.** The correct concurrency answer was given verbally at +44 min and never landed in a class, a signature, or a line of code.
9. **Speculative abstraction retained after being identified as speculative.** `RateLimitStrategy` + two named implementations, introduced before any logic was written, self-described as "for extensibility purposes, not to satisfy any current requirement" — and kept.

---

## Feedback Given

### Round Conditions

- **Hints used: 0/2.** No ceiling from hints. Everything given was a requirement answer, a request to justify his own claim, or a scenario handed over without explanation.
- **Requirements asked:** core operations · the legality rule · how the caller is identified and what the inputs are · what the caller sees when rate limited · (at +8 min, after phase 1 closed) whether limits are per-client.
- **Never asked:** lifecycle — entry expiry, map cleanup, reconfiguration mid-flight · concurrency posture · variants beyond per-client — tiers, multiple simultaneous limits, client cardinality · behaviour on an unknown or unconfigured client id.
- **Out of Scope list: produced unprompted.** One item ("distributed rate limiting"). It counts, and it's the item most people forget on this problem. It's also thin — persistence, client identification, and response headers were all sitting there.
- **Self-verified: no.** He submitted the implementation and stopped. The trace had to be requested. And the trace contradicts his own code.

### Walk Coverage

*(table as above — 4 Hit, 2 Partial, 2 Miss; dangling rules: none; silent assumptions table as above)*

### The Bug He Never Caught

Asked to trace `N=2, T=10`, client A, at `t = 0, 1, 5, 11`, his `t=11` answer was `clientState[A] -> {1,11}, allowed, quota: 0`. His code says `remove timestamps where timestamp <= now - T`. At `t=11`: `now - T = 1`, so the predicate is `timestamp <= 1`, removing `0` **and** `1`. List empties, `11` is pushed, real state is `{11}` with quota `1`.

| | Claimed trace | What the code does |
|---|---|---|
| `clientState["A"]` after t=11 | `{1, 11}` | `{11}` |
| quota left | `0` | `1` |

He traced the algorithm in his head, not the code he wrote — the exact failure mode a trace exists to catch, missed because he was confirming rather than executing. Plus: the method never returns its declared value on either path, `QuotaLeft` is never computed in the body, and `RateLimitManager` calls `getRateLimitForStatus` while the defined method is `getRateLimitStatus`.

### Rubric

- **Requirements & scoping — 3/5.** Four clean hits and an unprompted Out of Scope list — a real improvement on his usual opening. But the two Misses are the two that cost him: lifecycle (unbounded map) and concurrency posture (on a rate limiter). And the per-client question arrived at +8 min, after requirements were declared complete — that's patching, not scoping.
- **Entity modelling — 3/5.** Reasonable set, and he fixed it under questioning *without being told what was wrong*: dropped `Request` when he couldn't name what it did, moved per-client state into the strategy with the correct justification. Good judgement, costed by both fixes being reactive.
- **Class design — 2/5.** Interface declared a concrete implementation's field. Strategy method had no access to N and T until asked, and the fix was verbal, never written into the signature. The method name says *get status* and the method mutates — he confirmed he knew, and left the name.
- **Responsibility placement — 4/5.** Strongest technical dimension. Manager stays thin and delegates; state ended up owned by the thing that understands its shape; config-as-parameter put the right knowledge in the right place. No decision-making getters anywhere.
- **Implementation & correctness — 1/5.** No unprompted trace; prompted trace contradicted the code; declared return value never constructed on either path; no initialization for a first-time client; edge cases never enumerated (empty list, unknown client, N=0, T=0, clock skew).
- **Simplicity & judgement — 2/5.** Introduced `interface RateLimitStrategy` with two named implementations before writing a single line of logic, on a problem with exactly one rule. Asked which requirement drove it: *"It is for extensibility purposes. Not to satisfy any current requirement."* Honest, and the definition of a YAGNI violation — and having named it, he kept it. Restraint has been a consistent strength; it broke here, in the first thirty seconds of design.
- **Extensibility — 4/5.** Tier follow-up landed on a seam he'd already created — config as a parameter meant tiering was "resolve the right config and pass it," not a rewrite. On multi-limit he got the composition right, and when pointed at the partial-consume scenario he found the fix himself in one line: *"first the determine should be done then finally when both allow then go ahead and insert in both lists."* Check-then-commit, reached unaided. Held back only because his single mutating method is precisely the design that can't do that.
- **Concurrency — 5/5.** The best answer of the round and not close. Named the failure, named the **category** (correctness) before reaching for a primitive, identified the cause as concurrent access to shared state, proposed the coarse lock, **stated its cost** ("this will reduce throughput"), then refined to per-client lock striping. Category → primitive → cost → refinement, unprompted, in one paragraph. The senior answer.
- **Communication — 3/5.** Clear and responsive to pushback; engaged with substance instead of defending, and twice improved the design on the spot. Costs: left the "which language for phase 4" question unanswered across three turns while asking his own, and spent eleven minutes composing a one-sentence answer to the multi-limit follow-up.

### Pace Report

*(table as above)*

**Would this have fit a real 40-minute round? Marginally — no, but closer than he has been.** At 40 minutes he had finished the implementation and the first follow-up and was mid-way through composing the multi-limit answer. A real interviewer cuts him off **during follow-up two**. Lost: the check-then-commit insight, and the entire concurrency question — his 5/5. The single best evidence of senior capability in this round happens at minute 47 and would not exist in a real one.

**The good news is real:** requirements finished exactly on the 5-minute mark, and implementation came in *ahead* of reference. Those are the two phases he has historically blown. First LLD round where phase 1 didn't overrun.

**The time sink was one specific thing:** eleven minutes (10:56 → 11:07) to produce a single sentence on the multi-limit follow-up. Follow-ups are fast verbal exchanges — a senior answers "compose N independent limiters, allow only if all allow" in twenty seconds. This is a 3/5 problem, so the overrun is process, not problem size, and it was concentrated almost entirely in that one gap.

### Performance Rating: 2/5

This would have been a 3 — the entity model is sound, responsibility placement is genuinely good, the extensibility answers landed on real seams, and the concurrency answer is senior-grade. **Capped at 2: core logic with a flaw he never caught.** The trace contradicts his own eviction predicate at `t=11`, and the method never constructs the return value it declares. No hints used; no other ceiling applies.

### Senior Readiness Debrief

**1. Senior-signal scorecard**

| Signal | Status | Why |
|---|---|---|
| Scopes before designing | **Mixed** | 4/8 hit, Out of Scope unprompted, finished on the 5-min mark — but missed lifecycle and concurrency posture, and patched multiplicity in after phase 1 closed |
| State derived from requirements | **Mixed** | `clientState` correctly justified; `RateLimitStrategy` + two implementations admitted speculation; `QuotaLeft` never a requirement |
| Rules live with their state (Tell, Don't Ask) | **Strong** | Moved per-client state into the strategy on his own reasoning; manager stays a thin delegator; no decision-making getters |
| Simplicity held under pressure | **Weak** | Two strategy implementations before one line of logic, self-described as satisfying no requirement, and retained |
| Verifies own logic | **Weak** | No unprompted trace; the prompted trace contradicted the code; declared return value never built |
| Extends without rewriting | **Strong** | Tier follow-up landed on an existing seam; found check-then-commit unaided |

**Overall read: mid-level, with one clearly senior dimension.** Reasoning about *where things belong* and *what breaks under concurrency* is at the bar. Discipline about *what he actually wrote down* is not. Unusual profile — most people at this level have the opposite problem, and this one is the more fixable.

**Hire signal for this round: no-hire**, on the implementation phase alone. A rate limiter whose core method doesn't return its declared value, from a candidate who then traced it incorrectly, is not shippable — the interviewer's note reads "strong on design discussion, could not produce correct code." That is the exact note that ends senior loops.

**2. What a senior strong-hire would have done on THIS problem**

*The two requirement questions he skipped, and what they'd have changed:*
- *"Do client entries ever expire? What's my memory bound?"* — answer: unbounded distinct clients, untrusted traffic. That forces an eviction policy into the design at minute five — lazy cleanup on access plus a periodic sweep, or an LRU-bounded map. As designed, the rate limiter is a denial-of-service amplifier: an attacker sends one request per spoofed identifier and allocates a million lists that never free. Not a nitpick on a rate limiter; it *is* the failure mode.
- *"Is this shared across threads?"* — answer: obviously yes. Asked in phase 1, the per-client striped lock lands in the class design, in the signature, in the code. Asked at minute 44, it lands nowhere.

*The exact thing in the wrong class:* `clientState: Map<string, List<TimeStamps>>` declared on the `RateLimitStrategy` interface. He conceded immediately it's `SlidingWindowLog`'s — but the reason it matters is deeper: the moment that map is on the interface, every implementation is forced into "list of timestamps" as its state shape, making the promised `TokenBucket` (needing `tokens: double`, `lastRefill`) impossible. The abstraction added for extensibility is the thing blocking extension.

*The method that should have been two:* `getRateLimitStatus` reads a status and consumes a slot; he confirmed that's intentional. A senior splits it the moment the name and the effect disagree — `tryAcquire(clientId) -> Decision` (mutating) and `peek(clientId) -> Decision` (pure). And he already understands why: the multi-limit follow-up forced him to separate check from commit. Same split, discovered two phases later, under external pressure, for a case he thought was new. It wasn't new; it was the crack that was always there.

*The pattern he forced, and the one that would have paid:* Strategy with two implementations for one rule is the forced one. The pattern this problem earns is **Composite** — a `CompositeLimiter` holding a list of limiters, allowing only if all allow, committing only after all approve. He derived the need for it himself on the follow-up without naming it.

*The edge case a real trace catches:* not the `t=11` boundary, though that too. Run the code with `clientState` empty for a brand-new client — `remove timestamps from list`, which list? There is no entry. Any real trace starts at "first request from an unseen client" and hits the missing initialization in step one. He started at `t=0` and mentally supplied the empty list without noticing he'd supplied it.

**3. One concrete drill**

Fifteen minutes, as an executable, not on paper. Implement `SlidingWindowLog` for real in Python — a working `tryAcquire(clientId) -> (bool, int)`. Write exactly four assertions *before* running it:
1. First-ever request from an unseen client → allowed, quota `N-1`.
2. The `N=2, T=10` sequence at `t = 0, 1, 5, 11` — write down the *prediction* for `t=11` first, then run and compare. His prediction was wrong today; find out whether it's the code or the prediction to change, then write the boundary convention down as a requirement.
3. `N+1` requests in the same millisecond → exactly `N` allowed.
4. A client idle for `10T` → its list is empty or its entry is gone.

The point isn't the rate limiter — it's assertion 2: **predict, then execute, then compare.** He traced by narrating what he intended. A trace only has value when it can disagree with you, and the way to let it disagree is to commit the prediction in writing before stepping the code.

**4. Checklist**

`lld_senior_guidance.md`, the pre-round checklist. Two lines for a sticky note: *"Listed edge cases before being asked"* — never once done. *"Traced one concrete scenario out loud, naming state changes"* — done today, but only on request, and narrated rather than executed.
