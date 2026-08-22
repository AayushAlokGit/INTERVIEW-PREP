# Rate Limiter — Design Sheet

## 1. Requirements

1. **Operations**
   a. Check a request — given an incoming request, decide whether it is allowed through or rejected.
   b. Configure — a caller must be able to set up what the limit actually is before requests start flowing.

2. **Rules**
   a.A client may make at most N requests in any rolling window of duration T. Any window, not fixed calendar buckets. If a request would make it N+1 within the preceding T, that request is not allowed through.
   N and T are what the caller configures.
    b. Each client's budget rate limit budget tracked independently.

## Out of Scope

1. Distributed rate limiting.

## 2. Entities & Relationships

1. `Request(ClientID: string)`
2. `RateLimitManager(RateLimitStrategy: RateLimitStrategy, config: RateLimitConfiguration)`
3. `interface RateLimitStrategy` — implementations: `TokenBucket`, `SlidingWindowLog`
4. `RateLimitConfiguration(maxRequests: int, timeFrame: time)`

**Revised under questioning:**
- `Request` dropped — "does nothing", "no need for it".
- Per-client state is owned by the **strategy**, not the manager — "different states need to be stored for different algorithms".
- `RateLimitStrategy` interface retained for extensibility; acknowledged as not satisfying any current requirement.

## 3. Class Outlines

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

**Clarified under questioning:**
- `clientState: Map<string, List<TimeStamps>>` belongs to `SlidingWindowLog`, not the interface.
- Configuration is passed as a parameter to the strategy method by the manager.
- `getRateLimitStatus` is mutating: two back-to-back calls for the same client give different answers, "since each call would consume one quota".

## 4. Core Implementation (pseudo-code, as described)

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

## 5. Follow-ups (as answered)

**Tiers (free 100/min, premium 10k/min, enterprise 1M/hour):**
> "We would need to add a Client object since now each client has state of string + their tier. We would also need to add a configuration for each tier now. So the configuration would need to have a tier attribute as well. For each client when calling the strategy method we need to pass relevant config."

**Two simultaneous limits on one client (1000/min AND 20000/hour):**
> "we would need to store state for each limit separately and then apply the strategy on both state and if both states allow then the request is allowed."

**Partial-consume probe (per-minute allows and records, per-hour then denies):**
> "no first the determine should be done then finally when both allow then go ahead and insert in both lists"

**Concurrency (two threads, same client, one slot free):**
> "both the requests would be allowed and finally the list might grow to size 3 — a correctness issue. This is because of the concurrent access to the state. A solution for this is to serialise the access pattern; this can be done by introducing a lock when trying to get rate limit status for a request — this will reduce throughput. A better solution would be to take locks on the clientId so that different clientId requests can still be processed."

---
---

# Optimal Reference (what a senior strong-hire would design)

*This section is the teaching the round deliberately withheld. It includes everything that was never asked.*

## 1. Requirements + Out of Scope

**Functional**

- **FR1** — `tryAcquire(clientId)` returns a decision: allowed or denied, plus remaining quota.
- **FR2** — A client may make at most `N` requests in any *rolling* window of duration `T`. Not fixed calendar buckets.
- **FR3** — Budgets are per client and independent. Client A's traffic never consumes client B's allowance.
- **FR4** — `(N, T)` is configured by the caller before traffic flows.
- **FR5** — Being rate limited is a normal outcome, not an exception. Denial returns a value; only *programmer* errors (`N <= 0`, `T <= 0`) throw.
- **FR6** *(never asked)* — An unseen `clientId` is legal and starts with a full budget. There is no registration step.
- **FR7** *(never asked)* — Client entries must not accumulate without bound. An idle client's state is reclaimed. The number of distinct client ids is attacker-controlled.
- **FR8** *(never asked)* — The limiter is a **shared singleton across request threads**. Every mutation of shared state must be guarded.
- **FR9** *(never asked)* — Boundary convention, stated explicitly: a timestamp `ts` counts toward the window at time `now` iff `ts > now - T`. Exactly `T` old is expired. Write this down; it is the difference between two defensible traces.
- **FR10** *(never asked)* — Reads must be separable from writes. A dashboard asking "how much quota does A have left?" must not consume A's quota.

**Out of Scope**

1. Distributed / cross-node rate limiting (single process, in-memory).
2. Client identification — an opaque id arrives already extracted.
3. Persistence and restart survival — state is in-memory and lost on restart.
4. Response headers (`X-RateLimit-Remaining`, `Retry-After`) and HTTP concerns.
5. Metrics, logging, alerting.
6. Per-endpoint or per-route limits (single limit dimension for this design).

## 2. Entities & relationships

| Entity | Role |
|---|---|
| `RateLimiter` | **Orchestrator.** The single object callers talk to. Owns the clientId to bucket map, owns the lock striping, owns eviction. Delegates the *rule* downward |
| `LimitPolicy` | Immutable value object: `(maxRequests, window)`. Validated once at construction. Passed in, never mutated |
| `WindowState` (interface) | The **per-client** state for one algorithm. This is the correct seam — the varying thing is the *state shape plus its rule*, not a stateless helper |
| `SlidingWindowLogState` | Implements `WindowState`. Holds a deque of timestamps |
| `TokenBucketState` | Implements `WindowState`. Holds `tokens: double`, `lastRefill: Instant`. **Would not fit an interface that declares a timestamp list** |
| `Decision` | Immutable result: `(allowed: bool, quotaLeft: int, retryAfter: Duration)` |
| `Clock` | Injected time source. Not `now()` inline — otherwise the design is untestable |

The critical shape difference from the round: **the strategy is per-client state, not a per-limiter stateless service.** One `WindowState` object per client. That is what lets `TokenBucketState` and `SlidingWindowLogState` coexist, and it puts the lock exactly where the contention is.

## 3. Class outlines

```
interface WindowState:
    + tryConsume(now: Instant, policy: LimitPolicy) -> Decision   // mutates iff allowed
    + peek(now: Instant, policy: LimitPolicy) -> Decision         // pure
    + isIdle(now: Instant, policy: LimitPolicy) -> bool           // reclaimable?


class SlidingWindowLogState implements WindowState:
    - timestamps: Deque<Instant>

    + tryConsume(now: Instant, policy: LimitPolicy) -> Decision
    + peek(now: Instant, policy: LimitPolicy) -> Decision
    + isIdle(now: Instant, policy: LimitPolicy) -> bool
    - evictExpired(now: Instant, policy: LimitPolicy) -> void


class TokenBucketState implements WindowState:
    - tokens: double
    - lastRefill: Instant
    ... same three public methods


class LimitPolicy:                      # immutable value object
    - maxRequests: int
    - window: Duration

    + LimitPolicy(maxRequests: int, window: Duration)   // throws IllegalArgument if <= 0


class Decision:                         # immutable
    - allowed: bool
    - quotaLeft: int
    - retryAfter: Duration

    + static allow(quotaLeft: int) -> Decision
    + static deny(retryAfter: Duration) -> Decision


class RateLimiter:                      # ORCHESTRATOR
    - policy: LimitPolicy
    - clock: Clock
    - stateFactory: Supplier<WindowState>
    - buckets: Map<string, WindowState>
    - stripes: Lock[]                    # fixed array, e.g. 64

    + RateLimiter(policy: LimitPolicy, clock: Clock, stateFactory: Supplier<WindowState>)
    + tryAcquire(clientId: string) -> Decision      # mutating; the hot path
    + peek(clientId: string) -> Decision            # pure; safe for dashboards
    + evictIdle() -> int                            # returns entries reclaimed
    - lockFor(clientId: string) -> Lock


class CompositeLimiter:                 # the pattern this problem actually justifies
    - limiters: List<RateLimiter>

    + tryAcquire(clientId: string) -> Decision      # check ALL, then commit ALL
```

Note what is **not** here: no `Request` class, no `RateLimitManager` wrapping a `RateLimitStrategy` that owns every client's state, and no `getXxx` method that mutates.

## 4. Core method implementations

```python
class SlidingWindowLogState:
    def __init__(self):
        self.timestamps = deque()

    def _evict_expired(self, now, policy):
        cutoff = now - policy.window
        # FR9: exactly T old is expired; strictly newer than the cutoff survives.
        while self.timestamps and self.timestamps[0] <= cutoff:
            self.timestamps.popleft()

    def try_consume(self, now, policy):
        self._evict_expired(now, policy)
        if len(self.timestamps) >= policy.max_requests:
            # oldest entry frees a slot when it falls out of the window
            retry_after = (self.timestamps[0] + policy.window) - now
            return Decision.deny(retry_after)
        self.timestamps.append(now)
        return Decision.allow(policy.max_requests - len(self.timestamps))

    def peek(self, now, policy):
        # NOTE: still evicts, but never appends. Non-consuming from the caller's view.
        self._evict_expired(now, policy)
        remaining = policy.max_requests - len(self.timestamps)
        if remaining > 0:
            return Decision.allow(remaining)
        retry_after = (self.timestamps[0] + policy.window) - now
        return Decision.deny(retry_after)

    def is_idle(self, now, policy):
        self._evict_expired(now, policy)
        return len(self.timestamps) == 0


class RateLimiter:
    def __init__(self, policy, clock, state_factory=SlidingWindowLogState, stripes=64):
        self.policy = policy
        self.clock = clock
        self.state_factory = state_factory
        self.buckets = {}
        self.stripes = [Lock() for _ in range(stripes)]

    def _lock_for(self, client_id):
        return self.stripes[hash(client_id) % len(self.stripes)]

    def try_acquire(self, client_id):
        now = self.clock.now()
        with self._lock_for(client_id):                 # FR8: guards the whole check-then-act
            state = self.buckets.get(client_id)
            if state is None:                           # FR6: unseen client, full budget
                state = self.state_factory()
                self.buckets[client_id] = state
            return state.try_consume(now, self.policy)

    def peek(self, client_id):                          # FR10: read without consuming
        now = self.clock.now()
        with self._lock_for(client_id):
            state = self.buckets.get(client_id)
            if state is None:
                return Decision.allow(self.policy.max_requests)
            return state.peek(now, self.policy)

    def evict_idle(self):                               # FR7: bound the memory
        now = self.clock.now()
        reclaimed = 0
        for client_id in list(self.buckets.keys()):
            with self._lock_for(client_id):
                state = self.buckets.get(client_id)
                if state is not None and state.is_idle(now, self.policy):
                    del self.buckets[client_id]
                    reclaimed += 1
        return reclaimed


class CompositeLimiter:
    def try_acquire(self, client_id):
        now = self.clock.now()
        with self._lock_for(client_id):
            # PHASE 1 - check every limit, mutate nothing
            for lim in self.limiters:
                d = lim.peek_locked(client_id, now)
                if not d.allowed:
                    return d                            # no limit has been charged
            # PHASE 2 - all agreed; commit to all
            worst = None
            for lim in self.limiters:
                d = lim.commit_locked(client_id, now)
                worst = d if worst is None or d.quota_left < worst.quota_left else worst
            return worst
```

**Edge cases handled above, and the trace that would have caught each:**

| Edge case | Where |
|---|---|
| Unseen client | `try_acquire` creates the bucket lazily; `peek` returns full quota without allocating |
| `N` requests in the same instant | `>=` comparison plus append-after-check |
| Expired-exactly-at-`T` | `<= cutoff` per FR9, written down as a requirement rather than left implicit |
| Empty list on deny | Impossible — deny only occurs when `len >= N >= 1`, so `timestamps[0]` is safe |
| `N <= 0` / `T <= 0` | Rejected in `LimitPolicy`'s constructor, once, not on every call |
| Unbounded map | `evict_idle()`, plus `is_idle` on the state itself |
| Dashboard burning quota | `peek` is separate from `try_acquire` |

## 5. Design decisions, each against its alternative

| Decision | Alternative rejected | What it gives up |
|---|---|---|
| `WindowState` per client, not a shared stateless strategy | The round's `RateLimitStrategy` holding `Map<clientId, List<TimeStamp>>` | One small object per client instead of one big map. Buys: `TokenBucketState` becomes possible, and the lock sits on the client, not the algorithm. The round's shape *cannot* host a token bucket — its interface mandates a timestamp list |
| `tryAcquire` / `peek` split | One mutating `getRateLimitStatus` | Two methods instead of one. Buys: the name stops lying, and the composite's check-then-commit becomes expressible. The round's design discovered this need two phases later and could not satisfy it |
| Striped locks (fixed array) | One global lock; or one lock per client | Fixed array means two clients can collide on a stripe. Buys: bounded lock count (a per-client lock map is itself unbounded shared state needing its own lock — the problem recurses) with near-per-client concurrency |
| Injected `Clock` | `Instant.now()` inline | One constructor parameter. Buys: the `t=0,1,5,11` trace becomes an *executable test* rather than a paper exercise. This is the single change that would have caught this round's bug |
| Immutable `LimitPolicy`, validated once | `setRateLimitConfiguration(N, T)` on the limiter | Cannot reconfigure in flight. Buys: no torn reads of `(N, T)` mid-decision, no validation on the hot path, no "what happens to in-flight windows when T changes" question |
| **No pattern in the base design** | Strategy up front with two implementations | Nothing. One rule was specified; one implementation is written. `WindowState` is an interface because the *state shapes* genuinely differ, not because a second algorithm was imagined. **Composite** is the one pattern this problem earns — and only once multi-limit is a real requirement |

## 6. Extensibility — where each follow-up lands

| Follow-up | Seam | Change |
|---|---|---|
| **Tiers** (free / premium / enterprise) | `LimitPolicy` is already a parameter | Add `PolicyResolver: clientId -> LimitPolicy`, injected into `RateLimiter`. `try_acquire` resolves the policy per call. No entity changes, no state changes. *(This is the one the round got right — the seam existed because config was already a parameter)* |
| **Two limits at once** | `CompositeLimiter` wraps N limiters | New class, zero changes to `RateLimiter` or `WindowState`. Requires the `peek`/`commit` split to avoid partial consumption |
| **Swap to token bucket** | `state_factory` constructor param | `RateLimiter(policy, clock, TokenBucketState)`. `RateLimiter` never learns the algorithm exists |
| **Per-endpoint limits** | The key | Key on `(clientId, endpoint)` instead of `clientId`. Striping and eviction are unchanged |
| **Distributed** | Explicitly out of scope | `WindowState` becomes a Redis-backed implementation; the check-then-act moves into a Lua script. The interface survives; the lock does not |

## 7. Concurrency

**Category: correctness.** The failure is a classic check-then-act race — two threads read `len(timestamps) == 1` under `N = 2`, both conclude a slot is free, both append, the window now holds 3. Not coordination (nothing needs handoff or ordering) and not scarcity (there is no fixed pool being drawn down).

**Primitive: striped mutual exclusion.** The smallest thing that works, held across the *entire* read-decide-write sequence — evict, compare, append — because splitting it reintroduces the same race at a finer grain.

**Where it lives:** in `RateLimiter`, not in `WindowState`. The state objects stay plain and single-threaded by contract, which keeps them trivially testable; the orchestrator that owns the map owns the guarding. The map itself needs the same protection — lazy insertion of a new client is also check-then-act — which is why the lock is acquired *before* the `buckets.get`, not after.

**Cost:** clients hashing to the same stripe serialize unnecessarily (64 stripes, thousands of clients — collisions are frequent but each critical section is a few microseconds). Lock hold time is bounded by the eviction loop, which is amortized O(1) per call. No deadlock risk: exactly one lock, never nested. `CompositeLimiter` is the exception — it must hold one lock for the whole check-and-commit across all limiters, which is precisely why the composite holds the lock rather than each inner limiter taking its own.

**What was rejected:** a `ConcurrentHashMap` alone does not help — it makes `get` and `put` atomic individually, but the race is in the sequence between them. `computeIfAbsent` fixes only the insertion race, not the decide-then-append race inside the state. Atomics/CAS could work for a token bucket (a single counter) but not for a timestamp deque.
