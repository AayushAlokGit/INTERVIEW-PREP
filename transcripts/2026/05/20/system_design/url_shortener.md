# System Design Round Transcript
**Date:** 2026-05-20
**Start Time:** 9:51
**End Time:** 18:26
**Duration:** ~515 minutes (wall-clock; spans breaks — not continuous interview time)
**Problem:** Design a URL Shortener (TinyURL / bit.ly)

> **Integrity note:** The first set of answers to Probes 1-3 (deep dive) were flagged by
> Aayush as **not his own work**. Those answers were excluded, the three probes were
> **re-run**, and this transcript records his genuine answers. The feedback and scoring
> below reflect the re-run.

---

## Conversation Log

**Interviewer:** Presented the problem — design a URL shortener. Asked Aayush to note the
time, open the draw.io canvas, and start by gathering functional and non-functional
requirements with real numbers.

**Aayush:** Should we allow users to set custom aliases for URLs, and what about URL expiry?

**Interviewer:** Yes to both — custom aliases supported (generated if not provided),
optional TTL with a long-lived default. Asked for the rest of the FRs and the NFRs with
numbers.

**Aayush:** Is it fine to consider click analytics out of scope?

**Interviewer:** Yes — analytics out of scope. Good explicit scoping. Pushed for NFRs with
numbers.

**Aayush:** Is it fine to assume 10M DAU, most users redirecting rather than creating?
What read:write ratio should I consider?

**Interviewer:** That's a call you must make and justify, not ask for. Proposed he pick a
ratio and derive new URLs/day, write QPS, read QPS, storage over 5 years.

**Aayush:** 10M DAU, read:write 100:1. URL creation ~10M/100 = "1M urls/day", 100M
redirections/day. Write QPS ~1M/86400 ~ 10/s. Read QPS ~1000/s. Storage: 1M/day -> 365M/yr
-> ~1.8B in 5 years; ~2.5KB per record -> ~4.5TB -> ~5-6TB with buffer. "The numbers look
fine, let's move on to high level design."

**Interviewer:** Two corrections. (1) Arithmetic: 10M/100 = 100K, not 1M. (2) You skipped
latency and availability.

**Aayush:** NFRs: (1) 99.99% availability (~8.6 hrs downtime/year). (2) p99 < 100ms
redirect latency. (3) 10M DAU, read:write 100:1 -> 10M URLs/day ~100 writes/s, 1000M
redirects/day -> 10k reads/s.

**Interviewer:** Availability conversion is wrong — 99.99% is ~52 min/year; 8.6 hrs/year is
99.9%. Locked in ~100 w/s, ~10K r/s, p99 100ms, 99.99%. Asked for the high-level design.

**Aayush:** Gave APIs: (1) POST /shortUrls -> ShortUrl, body {longUrl, customAlias,
expirationTTL}. (2) GET /shortUrl -> LongURL.

**Interviewer:** APIs have gaps and aren't the architecture. Asked for explicit response
fields, the userId source, the 301 vs 302 trade-off, and the architecture diagram.

**Aayush:** See the draw.io file for a basic high-level architecture; it will evolve with
NFR considerations.

**Interviewer:** (Read diagram — Client -> API Gateway -> {Short Service, Redirect Service}
-> SQL DB; GET /shortUrl/:id -> 302.) Acknowledged the read/write split and 302 fix. Asked
the core question: how does Short Service generate the short code, guarantee global
uniqueness across instances, and handle custom aliases.

**Aayush:** Use an ID generator producing a unique ID (e.g. timestamp + machineId), then
base62-encode it. For custom alias, use the alias directly but check it doesn't exist via
cache/bloom filter. (Follow-up:) SHA-256 hashing gives random codes but loses ordering; a
monotonically increasing counter preserves ordering and DB locality. ~1B URLs/year, so
encode the counter in base62 (5 chars ~ ~1B values). For multiple instances, coordinate
the global counter via Redis atomic INCR; reduce Redis load by handing out fixed-size
batches/ranges — each instance processes its batch then requests another.

**Interviewer:** Strong answer. Posed three probes: (1) Redis on the critical write path —
what on failure, and counter gaps on crash? (2) Custom alias colliding with a future
counter-generated code — how to prevent? (3) Bloom filter "maybe" — next step, and why
keep it?

**[Probe responses round 1 — EXCLUDED]:** A set of answers to all three probes was
submitted but subsequently flagged by Aayush as not his own work. They have been removed
from assessment.

**Interviewer:** Re-ran all three probes, asking Aayush to answer in his own words.

**Aayush (Probe 1, genuine):** Redis cluster with master-slave for HA. A circuit breaker
in front of Redis prevents further URL-shortening requests when Redis truly goes down —
effectively denying service for writes during a sustained outage. If a Short Service
instance crashes mid-batch, that batch is discontinued; on recovery it requests a new
batch. Continuity breaks but codes are still monotonically increasing.

**Interviewer:** Solid. The batch allocation already cushions a *transient* outage
(instances keep serving from their local batch); the circuit breaker is for a *sustained*
outage — distinguish the two. Crash/gap reasoning correct.

**Aayush (Probe 2, genuine — attempt 1):** Use customAlias only during the redirect flow —
the URL gets a generated short code regardless, and the customAlias is also respected.
Enforce a uniqueness constraint on the customAlias column at the DB level; bloom filters
can help users pick unused aliases.

**Interviewer:** That does not solve it. A unique constraint on the customAlias column only
stops two custom aliases clashing — it does nothing to stop a custom alias equaling a
generated code. A redirect for "dribble" where "dribble" is both a generated code and a
custom alias is still ambiguous.

**Aayush (Probe 2, genuine — attempt 2):** For the customAlias, check if a record already
exists with that short code; if so, assign that record a new id and use the custom alias
as the id for the current record.

**Interviewer:** Two problems. (1) The collision is created later by the counter, not at
alias-creation time — your check looks at the wrong moment. (2) You cannot reassign an
already-issued short code; its links are already shared, reassigning breaks them. Revealed
the two correct mechanisms: (a) disjoint-by-length — fixed-length generated codes (7),
custom aliases any length except 7; (b) one unified keyspace with a single unique-key
column, where a counter-generated insert that collides is rejected and the counter
advances.

**Aayush (Probe 3, genuine):** Not sure about the bloom filter — doesn't know much about
it.

**Interviewer:** Explained the bloom filter: bit array + k hash functions; "definitely not
present" or "maybe present"; no false negatives, possible false positives. For the alias
check — "definitely not present" skips the DB; "maybe present" falls back to a DB lookup.
Worth keeping because it resolves the common (not-taken) case in-memory and shields the DB;
reliable only in the "no" direction.

**Interviewer (earlier, read path):** Asked the read path — 10K reads/s, p99 100ms.

**Aayush (read path, genuine):** Add a Redis cache between Redirect Service and DB. Check
cache first; hit -> return long URL; miss -> fetch from DB and populate the cache.
Immutable URLs -> low staleness -> high TTL. LRU eviction keeps the hot set cached; hit
ratio is high due to an 80/20 access pattern. Background cleanup removes expired entries
from cache and DB.

**Interviewer:** Complete and correct read path. Correction: in cache-aside the Redirect
Service reads the DB on a miss and writes the cache — Redis does not fetch from the DB
itself. Wrapped up.

---

## Design Summary

**Requirements Gathered:**
- Functional: shorten a long URL (generated or custom alias), redirect short -> long,
  optional TTL/expiry, link deletion. Out of scope: click analytics, URL versioning.
- Non-functional: 99.99% availability; p99 < 100ms redirect latency; 10M DAU, read:write
  100:1; ~10M URLs/day (~100 writes/s), ~1B redirects/day (~10K reads/s); ~1.8B URLs and
  ~5-6TB storage over 5 years.
- Errors made: 10M/100 computed as 1M (should be 100K); 99.99% labeled ~8.6 hrs/year
  (should be ~52 min/year).

**High-Level Architecture:**
- Client -> API Gateway -> Short Service (writes) and Redirect Service (reads) -> SQL DB.
- Redis as the global counter authority (atomic INCR, batch/range allocation).
- Redis cache on the read path (cache-aside) between Redirect Service and DB.
- Data model: ShortUrl {id, longUrl, userId, createdAt, TTL, customAlias}, User {id, name}.

**Key Design Decisions & Trade-offs:**
- Separate read/write services for independent scaling.
- Monotonic counter + base62 over hashing — preserves ordering and DB locality.
- Redis INCR with batch allocation — reduces per-write Redis load.
- 302 redirect (not 301) — avoids browser caching, keeps control server-side.
- High TTL + LRU cache justified by URL immutability and 80/20 access pattern.
- Master-slave Redis + circuit breaker for write-path resilience.

**Scalability & Fault Tolerance Points (genuine):**
- Master-slave Redis with circuit breaker; sustained Redis outage degrades write
  availability (fail-fast), read path unaffected.
- Counter gaps from instance crashes acceptable (uniqueness, not continuity, required).
- Cache-aside read path absorbs 10K reads/s; background cleanup of expired entries.

**Gaps / Missed Areas:**
- Custom-alias vs generated-code collision — not solved across genuine attempts;
  mechanisms had to be revealed.
- Bloom filter — knowledge gap; had to be explained.
- No load balancer in design or diagram.
- Read-path cache described verbally but not drawn (Redis wired only to the counter path).
- API response shape never enumerated; auth/userId source and create-idempotency not
  addressed.
- DB scaling for ~1.8B rows / 5-6TB not proactively discussed.
- Duplicate "Short Service" box left on the diagram.

---

## Feedback Given

Re-scored on Aayush's genuine answers; the disputed Probe 1-3 responses were excluded.

| Criterion | Score | Notes |
|---|---|---|
| Requirements Clarification | 6/10 | Good scoping questions; tried to skip latency + availability; arithmetic slips (10M/100, 99.99% conversion). |
| High-Level Architecture | 6.5/10 | Clean Client -> API Gateway -> split Short/Redirect services -> DB; no load balancer; answered "APIs" when asked for architecture. |
| API Design | 5.5/10 | POST body fine; response fields never enumerated despite asks; userId/auth source and create-idempotency unaddressed. |
| Component Design & Trade-offs | 5.5/10 | ID generation (counter + base62 + Redis batch allocation) strong and his own. Custom-alias collision — three genuine attempts, no working mechanism reached. |
| Scalability & Fault Tolerance | 7/10 | Probe 1 genuine and good: master-slave Redis + circuit breaker + correct crash/gap reasoning. Read-path cache solid. |
| Deep Dive Quality | 5/10 | ID-gen dive good. Probe 2 (namespace disjointness) not solved. Probe 3 (bloom filter) — knowledge gap. |
| Communication | 6/10 | Honest about not knowing bloom filters. Diagram lags the verbal design. |

**Overall: ~6.0/10**

### Diagram Quality
- Present: Client, API Gateway, Short Service, Redirect Service, SQL DB, Redis, a clear
  data model, and FR/NFR/API annotations. Data flow directional and labeled.
- Gaps: read-path cache described verbally but not drawn (Redis wired only to the counter
  path); no load balancer; duplicate "Short Service" box left on canvas.

### Top things to fix
1. Stop fast-forwarding past NFRs — state all NFRs with numbers before architecture.
2. Know availability math cold: 99.9% = 8.76 hrs/yr, 99.99% = 52 min/yr, 99.999% = 5 min/yr.
3. Namespace collisions — when two generators share a keyspace, make collision impossible
   by construction (disjoint keyspaces or a unified unique-keyed table), don't patch it
   after the fact.
4. Learn bloom filters — they recur across system design (shorteners, crawlers, DBs, caches).
5. Keep the diagram in sync with the spoken design.

### Genuine strengths
ID-generation deep dive (counter + base62 + Redis batch allocation, with DB-locality
reasoning) was strong and self-driven. Resilience instincts are real — master-slave Redis,
circuit breaker, correct crash/gap reasoning. Reasoning for component choices was stated
throughout, and he was honest about a knowledge gap rather than bluffing.
