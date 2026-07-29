# System Design Round Transcript
**Date:** 2026-06-07
**Start Time:** 10:37
**End Time:** 12:23
**Duration:** 106 minutes
**Problem:** Design an E-Commerce Order Management System

---

## Conversation Log

**Interviewer:** Presented the problem — design the backend powering checkout and order lifecycle for a large online retailer (Amazon-scale); the system owns the order from "Place Order" through a terminal state. Asked Aayush to start with requirements (functional + non-functional) and scale.

**Aayush:** What can the scale of the system be?

**Interviewer:** Pushed back — wants him to propose the numbers himself (deriving scale is part of the evaluation). Anchored: large retailer, pick daily orders and work out orders/sec avg & peak, read/write ratio, storage growth.

**Aayush:**
- FRs: browse catalog, add to cart, place order + payment, order status tracking, view orders. Out of scope: analytics, ratings.
- NFRs: highly available browsing (99.99%); strong consistency for orders (no multiple orders for same item); no double charging; status event → DB update < 1 min; 100M DAU, 2 orders/day → 2k writes/s avg, 20k/s peak; reads 5×/order → 100k reads/s peak; 100 KB/order → 20 TB/day.

**Interviewer:** Sanity-checked the math (2.3k/s avg ✓, 10× peak ✓). Flagged 100 KB/order as large (orders are usually a few KB); asked if intentional. Asked whether NFR #2 means inventory correctness (no overselling). Noted catalog browse is likely the dominant read path and durability target for orders was missing.

**Aayush:** Revised order size to 10 KB → 2 TB/day. Added catalog scale: 1M item catalog, 50 browses/user/day → 50k item reads/s avg, 500k/s peak.

**Interviewer:** Accepted. Moved to core entities.

**Aayush:** Item (id, name, description), ItemInventory (itemId, count), Order (id, userId, createdAt, status, amount), OrderItem (orderId, itemId, itemCount, price), User (id, name, email). Noted he'd iterate.

**Interviewer:** Credited the Item/ItemInventory and Order/OrderItem splits. Pointed out missing Cart entity (despite FR) and missing Payment entity (despite no-double-charge NFR).

**Aayush:** Confirmed there'd be Cart and Payment entities; order id linked to payment.

**Interviewer:** Asked for Payment's key fields and the order↔payment relationship.

**Aayush:** Payment (id, amount, method, idempotencyKey, createdAt, status, recipient, sender).

**Interviewer:** Credited the self-raised idempotencyKey. Moved to API design.

**Aayush:** (identity from auth token)
- GET /items?query&cursor&limit → [Item(id,name,description,quantity)]; cursor pagination to handle dynamic catalog inserts.
- POST /carts req [Item] → 2xx.
- PUT /carts/:id req [{add/remove, itemId}] → 2xx.
- POST /orders req {[CartItems], Address, PaymentDetails} → OrderId + 2xx (PAYMENT_PENDING); Idempotency-Key header.
- PUT /orders/:id req {status} → 2xx.
- GET /orders/:id → Order(id, status, items[], createdAt).
- GET /orders → Order(id, status, createdAt)[].

**Interviewer:** Credited cursor pagination w/ justification and the idempotency key. Three probes: (1) GET /orders has no pagination; (2) PUT /orders/:id {status} lets any client set arbitrary status / invalid transitions; (3) payment flow hand-wavy — sync vs async charge, where idempotency stops double-charge.

**Aayush:** (1) Add cursor pagination to GET /orders too. (2) Status updates should be internal-only, no client-facing API. (3) Remove PaymentDetails from order request; create order PAYMENT_PENDING, prompt user, user pays, trigger 3P provider, provider notifies us on completion, update DB state.

**Interviewer:** Accepted. Banked two threads for deep dive: delayed/never-arriving provider callback, and duplicate-callback dedupe. Moved to high-level design (browse + place-order/payment flows).

**Aayush:** (asked to add core entities + API to the diagram while explaining HLD)

**Interviewer:** Updated the diagram. [Aayush then self-corrected that Payment needs an orderId; interviewer added it.]

**Aayush (HLD - read path):** Client → API Gateway (auth, future rate limiting) → Catalog Service → Postgres (chose Postgres for inverted-index support; read-heavy traffic). Single DB instance can't meet 500ms latency at scale; read replicas still bottleneck; add in-memory cache with LFU eviction so popular queries hit cache, less-popular hit DB.

**Interviewer:** Probed: (1) named alternative — why Postgres over Elasticsearch for full-text search at 500k/s; is a CDN viable since catalog is mostly static? (2) staleness trap — GET /items returns quantity, which changes on every purchase; how to avoid stale stock in the cache?

**Aayush:** (1) Use Elasticsearch; keep it in sync via CDC events from the SQL DB on catalog changes, removing the DB from the read path (DB = source of truth). (2) Don't cache quantity; serve it live as part of the individual item query.

**Interviewer:** Accepted. Moved to the write path; flagged the key question: reserve inventory without overselling when 10k race for the last units.

**Aayush (asked interviewer to draw read path while he explained write path).**

**Aayush (write path):** Client adds to cart (creates cart if none; cart unique on userId; persisted in DB for cross-device). Place order for cart or single item. POST with order items + delivery details → OrderService creates order records PAYMENT_PENDING; idempotent via idempotency-key header. User led through payment; Payment record created; payment status used as idempotency to avoid double-charge while one is in progress. On completion, listen to payment-provider notifications → update order & payment state. Post-payment lifecycle: PAYMENT_DONE → Delivery Scheduled → Picked Up → Delivered; cancellable before pickup. Modeled lifecycle as a long-running workflow listening to external events (e.g. cancellation).

**Interviewer:** Credited the workflow modeling. Flagged that inventory reservation was skipped (asked twice). Deep-dive questions: (1) when is stock decremented — at order creation or after payment, trade-offs? (2) concurrency — how does exactly one of two racers win? (3) reservation lifetime — release if user never pays?

**Aayush:** Acquire a distributed lock with TTL on item inventory at order creation; DB constraint inventory ≥ 0; acknowledged contention reduces write throughput; suggested optimistic concurrency control via a version on ItemInventory to improve throughput.

**Interviewer:** Pressed: (1) why a distributed lock when an atomic conditional decrement (`UPDATE inventory SET count=count-:qty WHERE item_id=:id AND count>=:qty`) prevents oversell with no external lock — what does the distributed lock buy, what failure modes does it add? (2) scale break — flash sale, 10k checkout/s on the same inventory row: why do both pessimistic and optimistic degrade, and how to serve one hot item at that rate?

**Aayush:** (1) With DB row-level locking, distributed locks aren't needed. (2) The 10k requests serialize → write throughput degrades. At 20k orders/s, use an in-memory locking mechanism to offload the DB — same operations as the DB transaction but inside an atomic Redis Lua script.

**Interviewer:** Credited Redis/Lua (~100k ops/s on a single key handles 10k/s where a Postgres row chokes). Two follow-ups: (1) durability & reconciliation — Redis is now authoritative and crashes mid-sale; source of truth? how do Redis and DB stay consistent and correct? (2) reservation release — decrement at order creation, user abandons; how to release the unit?

**Aayush:** Write Redis data to DB asynchronously; fall back to DB transactions when Redis is down; run reconciliation jobs to keep DB and Redis in sync. For release: a TTL mechanism that auto-increments the inventory counter on expiry.

**Interviewer:** Noted both were directionally right but thin (async writes lose the last second on crash; DB fallback can't take the 10k/s; reservation needs a per-reservation record + sweeper/keyspace-notification). Wrapped up and delivered feedback + senior debrief.

---

## Design Summary

**Requirements Gathered:**
- FRs: browse catalog, cart, place order + payment, status tracking, view orders. Out of scope: analytics, ratings.
- NFRs: 99.99% browse availability; strong consistency / no oversell; no double charge; status→DB < 1 min; 100M DAU × 2 orders/day → 2k/s avg, 20k/s peak writes; order reads 100k/s peak; 10 KB/order → 2 TB/day; 1M catalog, item reads 50k/s avg, 500k/s peak.

**High-Level Architecture:**
- Read path: Client → API Gateway (auth, future rate limit) → Catalog Service → LFU query cache; Elasticsearch search index kept in sync via CDC from Postgres (source of truth); live `quantity` read from DB on item-detail path.
- Write path: Cart (unique per user, persisted) → OrderService creates order PAYMENT_PENDING (idempotency-key); Payment record (idempotency via payment status); async 3P payment provider with completion callback updating state; post-payment order lifecycle modeled as a long-running workflow.
- Inventory: DB row-level atomic decrement with `count >= qty` guard; under flash-sale load, move the hot counter into Redis with atomic Lua; async write-back + reconciliation; TTL-based reservation release.

**Key Design Decisions & Trade-offs:**
- Cursor pagination over offset (dynamic catalog inserts).
- Elasticsearch (via CDC) over Postgres on the read path; Postgres as source of truth.
- Don't cache `quantity` (avoid stale stock).
- Row-level atomic decrement over distributed lock (lock redundant); Redis/Lua to escape single-row serialization at flash-sale scale.
- Order lifecycle as a long-running, event-driven workflow.

**Scalability & Fault Tolerance Points:**
- Read replicas → cache (LFU) → ES to remove DB from read path.
- Recognized single-row serialization as the scale break; Redis/Lua to absorb hot-item throughput.
- Reconciliation jobs and TTL reservation release (thin).

**Gaps / Missed Areas:**
- Did not self-raise the inventory/oversell problem (the core challenge) — needed two flags.
- Single-Redis-key ceiling / inventory sharding not raised.
- Reservation not modeled as a first-class entity; release mechanism hand-waved.
- Payment robustness: delayed/never-arriving callback (reconciliation poll + timeout), duplicate-callback dedupe, outbox pattern — not covered.
- Operability: monitoring/oversell-detection, consumer lag, cost — not covered.
- Pace: 106 min, well over the 45–50 min budget.

---

## Feedback Given

### Standard Evaluation
- **Requirements clarification — Strong.** FRs + NFRs with avg & peak numbers he derived himself (improvement); revised order size and added catalog sizing under challenge. Durability and catalog-read volume needed prompting.
- **Core entities — Strong.** Item/ItemInventory and Order/OrderItem splits; Payment with idempotencyKey; self-corrected Payment.orderId. Cart/Payment only surfaced when asked.
- **API design — Strong (most-improved).** Cursor pagination with justification, idempotency key unprompted, clean corrections (pagination on GET /orders, internal-only status, fixed payment flow). Some bare-2xx response shapes.
- **High-level architecture — Strong on read path.** Gateway → catalog → Postgres → replicas → LFU cache → ES via CDC; pulled quantity out of cache to avoid stale stock.
- **Component design & trade-offs — Mixed.** Committed row-lock over distributed lock; volunteered OCC trade-off; ES alternative came only after prompting.
- **Scalability & fault tolerance — Mixed.** Reached Redis/Lua for the hot row; durability, reconciliation, reservation release stayed thin.
- **Deep dive — Mixed.** Walked past the core oversell problem twice; progressed once flagged but I drove each escalation; back half hand-waved.
- **Communication — Good when engaged**, thinned toward the end; used "draw while I talk" three times (partly efficient, partly deferring).

### Senior-Signal Scorecard
| Signal | Rating | Reason |
|---|---|---|
| Owns the narrative (self-raises traps) | Mixed | Self-raised idempotency & Payment.orderId; missed central oversell trap until flagged twice |
| Leads with trade-offs vs alternatives | Mixed | Volunteered OCC vs pessimistic; ES alternative only after I named it |
| Pushes scale until it breaks | Mixed | Handled hot-row break well but I drove every escalation |
| API as a designed contract | Mixed (↑ from Weak) | Cursor pagination + idempotency unprompted; some bare-2xx shapes |
| Operability / second-order concerns | Mixed (↑ from Weak) | Raised reconciliation & TTL release; missed monitoring, callback-failure, cost |
| Pace | Weak | 106 min vs 45–50 target; core locked very late |

**Overall level read:** Mid-level with clear senior flashes — **hire, not strong-hire.**

### What a Senior Strong-Hire Would Have Done on THIS Problem
- Named inventory consistency under concurrency as THE challenge during requirements, not after two nudges.
- Decided reserve-at-creation vs reserve-at-payment explicitly with trade-offs.
- Gone to the atomic conditional decrement, then self-raised the hot-row/single-key ceiling and proposed inventory sharding / counter bucketing.
- Modeled the reservation as a first-class entity (reservationId, expiresAt) with sweeper / keyspace-expiry restore and saga compensation on payment failure.
- Owned payment robustness: at-least-once webhook dedupe, reconciliation poll + payment-pending timeout for missing callbacks, outbox pattern for state transitions.
- Covered operability: oversell-detection alerting, consumer lag, Redis fleet cost.
- Locked core by ~30 min, spent the back half on the inventory deep dive.

### Self-Drill
Review `system_design_senior_guidance.md`, especially #1 (self-raise traps), #3 (push scale yourself), #6 (pace). API discipline (#4) visibly improved.

**Time Taken: 106 minutes**
