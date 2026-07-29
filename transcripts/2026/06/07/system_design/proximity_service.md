# System Design Round Transcript
**Date:** 2026-06-07
**Start Time:** 16:28
**End Time:** 17:18
**Duration:** 50 minutes
**Problem:** Design a Proximity Service (Find Nearby Places / Yelp)

---

## Conversation Log

**Interviewer:** Presented the problem — backend for "find places near me" (Yelp/Google Maps nearby). Users search by (lat, lng, radius); owners CRUD listings. Asked Aayush to gather requirements with numbers, aiming for a senior pace.

**Aayush:** Do we also need to show the menu of local restaurants, or just the list?

**Interviewer:** Scoped to discovery only — return list with lightweight metadata; menus/reviews/photos out of scope. Two things in scope: search and management.

**Aayush:** What scale — 10M DAU and 10M restaurants?

**Interviewer:** Suggested 10M DAU and ~100M places; asked him to derive searches/sec, writes/sec, storage, and NFRs himself.

**Aayush:** FRs: (1) query restaurants within a radius of location; (2) owners CRUD restaurants. Out of scope: menu & ordering. NFRs: (1) HA querying 99.9%; (2) p99 < 200ms; (3) eventual consistency OK for listing updates; (4) 10M DAU, 100M restaurants, 3 queries/user/day, 10 results/query → ~3k restaurant reads/s avg, 30k/s peak, high read load.

**Interviewer:** Flagged the conflation: 10M × 3 = 30M searches/day ≈ 300 searches/s avg (~3.5k peak); the ×10 is records returned, not request rate. Asked which number drives capacity. Also asked for write rate and storage.

**Aayush:** Updated: 300 queries/s. Storage: 10KB/restaurant × 100M = 1TB.

**Interviewer:** Asked for write rate explicitly.

**Aayush:** Assume 1 update/day per restaurant listing. Then asked interviewer to add FRs/NFRs to the diagram.

**Interviewer:** Flagged that 1 update/day × 100M ≈ 1,150 writes/s, which exceeds the 300 searches/s — contradicts "read-heavy"; realistically owners update far less often. Added FR/NFR boxes to the diagram.

**Interviewer:** Moved to core entities.

**Aayush:** Restaurant (id, lat, lng, name, description, ownerId), User, Owner.

**Interviewer:** Suggested adding category + rating (for filters) and a geohash index key.

**Aayush:** Agreed — add category, rating, and geohash for coordinates to help location querying.

**Interviewer:** Moved to API design.

**Aayush:** (identity from auth header) POST /restaurants {name, description, lat, lng, ...} → restaurantId; GET /restaurants?lat&lng&query&radius → Restaurant(id, name, rating, category, ...)[]; PUT /restaurants/:id {name, description} → 2xx.

**Interviewer:** Probed: pagination + ordering on search (dense cities); missing DELETE; explicit response fields (distance/lat/lng); server-generated id + idempotency on create.

**Aayush:** Revised: added Idempotency-Key header on POST; cursor pagination + limit on GET; lat/lng in response; added DELETE /restaurants/:id.

**Interviewer:** Noted cursor pagination on a spatial query needs a defined sort order (distance). Asked him to add entities + API to the diagram, then moved to HLD — specifically how geohash search works.

**Aayush (HLD):** Owners hit POST/PUT/DELETE → API service writes to PostgreSQL. Read path: API queries DB for restaurants whose lat/lng fall within radius — naive, requires table scan. Use PostGIS extension with geospatial indices to make it efficient. But at this read scale the DB becomes a bottleneck; read replicas help but 3k/s peak is hard, and querying 100M restaurants adds latency. Better: precompute restaurants per region in Redis, key = geohash of region, value = list of restaurants; API hashes user location and returns restaurants, eliminating DB from the hot path. Cache updated on CRUD. No sharding needed (small data). Core problem = low-latency location range queries.

**Interviewer:** Probed the geohash mechanism — Problem 1: (a) a single cell isn't the radius (1km vs 50km), (b) boundary case where closest restaurant is just across the cell edge.

**Aayush:** (b) Get all neighboring geohashes of the user's cell to cover boundaries. (a) Fix the per-geohash area, compute the required regions from radius + location, pick the covering geohashes.

**Interviewer:** Problem 2 (scale break) — fixed cell size: Manhattan cell ~5,000 restaurants (huge value + hot key), rural cell 2; shrinking cells breaks rural/large-radius queries. How to resolve the tension?

**Aayush:** Not sure how to optimize further.

**Interviewer:** Hint — the grid is fixed; what if a crowded cell could split and a sparse area stayed coarse? What structure subdivides recursively only where dense (like storing a map/image with uneven detail)?

**Aayush:** A quadtree.

**Interviewer:** Asked him to develop it: leaf rule, search traversal, and operability (where it lives, build, updates, recovery).

**Aayush:** A node splits when it reaches a restaurant threshold. Admitted not very familiar with quadtrees.

**Interviewer:** Confirmed the leaf rule, said no deep internals needed; explained search descends and prunes by bounding-box intersection. Asked the operability three: build/source of truth, how sparse writes reach in-memory quadtrees, recovery on restart.

**Aayush:** (1) Update the index while writing the restaurant to the DB. (2) CDC events from the DB trigger index updates; could add Kafka for durability but it's unnecessary complexity at this scale. (3) Search servers flush the index to local disk to recover. Also: shard the restaurant index by region so searches route to relevant regions; search servers behind a load balancer.

**Interviewer:** Credited the trade-off judgment and the self-raised geographic sharding. Probed two consequences: (1) hot-region load skew (Manhattan shard ≫ Montana); (2) cross-shard boundary queries.

**Aayush:** (1) Duplicate hot-region data across multiple shards/nodes; query service randomly picks one — introduces consistency-maintenance complexity. (2) The API service performs the joins/merge across the regions.

**Interviewer:** Final probe — how do you know a search server's quadtree has gone stale/diverged (CDC consumer lagged/died)?

**Aayush:** Put CDC events through Kafka, partition by region, track consumer-producer lag as a measure of index staleness.

**Interviewer:** Credited using lag as a staleness signal and the revision of the earlier Kafka call. Wrapped up and delivered feedback + senior debrief.

---

## Design Summary

**Requirements Gathered:**
- FRs: radius search for restaurants; owner CRUD on listings. Out of scope: menu/ordering.
- NFRs: 99.9% availability, p99 < 200ms, eventual consistency for listings; 10M DAU, 100M restaurants, ~300 searches/s avg (~3k peak), 10 results/query, ~1TB storage; writes ~1/day/restaurant (flagged as inconsistent with read-heavy).

**High-Level Architecture:**
- Write path: Owner → API service → PostgreSQL (source of truth); index updated via CDC.
- Read path: precomputed per-region restaurant lists in Redis keyed by geohash; API hashes user location (+ neighbor cells, covering set for radius) and returns results; DB off the hot path (PostGIS as fallback/source).
- Density handling: adaptive quadtree (split leaf on restaurant threshold) reached with a nudge.
- Sharding: geographic (by region) behind a load balancer; hot regions replicated; cross-shard boundary queries merged by the query/API service.
- Operability: local-disk snapshot for recovery; CDC via Kafka partitioned by region; consumer lag as staleness signal.

**Key Design Decisions & Trade-offs:**
- PostGIS spatial index over naive scan; Redis precompute over DB-on-hot-path (latency at scale).
- Kafka judged overkill initially, then adopted for lag-based staleness monitoring (trade-off revised).
- Hot-region replication accepted at the cost of copy consistency (fine given eventual consistency NFR).

**Scalability & Fault Tolerance Points:**
- DB read bottleneck identified and removed via Redis precompute.
- Density skew → quadtree (with help).
- Geographic sharding, hot-region replication, cross-shard scatter-gather + merge.
- Recovery via local snapshot; staleness via consumer lag.

**Gaps / Missed Areas:**
- Did not self-raise the density-skew trap (interviewer surfaced it); blanked before reaching quadtree.
- Limited depth on quadtree internals / didn't compare geohash vs quadtree vs S2/H3 with trade-offs.
- Cross-shard distance-sorted cursor pagination not addressed in depth.
- Write-rate assumption inconsistent with "read-heavy"; not reconciled.
- Diagram captured requirements/entities/API only; the HLD (Redis/quadtree/sharding) was described verbally, not drawn.

---

## Feedback Given

### Standard Evaluation
- **Requirements — Strong.** Self-proposed scale, good scoping question, corrected request-rate vs result-volume after one prompt. Soft spot: write-rate assumption contradicts read-heavy, not reconciled.
- **Core entities — Good.** Restaurant + geohash/category/rating (geohash instinct his own).
- **API — Strong.** Complete after one prompt: cursor pagination, idempotency, explicit fields, DELETE.
- **HLD — Strong.** Self-drove naive → PostGIS → bottleneck → Redis geohash precompute.
- **Component design & trade-offs — Strong.** Kafka judged then revised; replication with consistency cost.
- **Scalability & fault tolerance — Strong.** Geographic sharding, hot-region replication, cross-shard merge, snapshot recovery.
- **Deep dive — Mixed.** Density skew not self-raised; blanked then reached quadtree with a nudge; honest about geo knowledge gap.
- **Communication — Strong.** Concise, on-pace, honest about knowledge boundary.

### Senior-Signal Scorecard
| Signal | Rating | Reason |
|---|---|---|
| Owns the narrative (self-raises traps) | Mixed (improved) | Drove read-path break & sharding; density trap needed surfacing |
| Leads with trade-offs vs alternatives | Strong (improved) | PostGIS/Redis/Kafka/replication justified; revised Kafka call |
| Pushes scale until it breaks | Mixed | Broke DB read path himself; couldn't push past density break unaided |
| API as a designed contract | Mixed | Complete contract but pagination/DELETE/idempotency prompted |
| Operability / second-order concerns | Mixed (improved) | Recovery, CDC, lag-as-staleness, hot-region — mostly when asked |
| Pace | Strong (much improved) | 50 min, core locked early (vs 106 earlier today) |

**Overall level read:** Borderline strong-hire — strongest round of the day. Blockers from a clean strong-hire: not self-raising the density-skew trap, and a knowledge wall on spatial indexes.

### What a Senior Strong-Hire Would Have Done on THIS Problem
- Self-raised the density skew during HLD ("fixed geohash grid breaks under uneven density → adaptive index").
- Known the geo toolkit (geohash vs quadtree vs S2/H3) and chosen with trade-offs; named that Uber/Google use S2/H3 for this.
- Detailed quadtree leaf rule, search traversal (descend + prune by box intersection), and write-time split/rebuild.
- Owned the hard cursor: distance-sorted, scatter-gathered pagination needs a stable distance+id cursor across shards.
- Noted eventual consistency makes hot-region geo-replication / edge caching cheap for global p99.

### Self-Drill
1. Learn the geo-index family (geohash, quadtree, S2/H3) well enough to compare with trade-offs.
2. Keep drilling guidance #1 (self-raise the trap); pace (#6) and trade-offs (#2) are now strong — bank them.

**Time Taken: 50 minutes**
