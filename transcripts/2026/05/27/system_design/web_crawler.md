# System Design Round Transcript
**Date:** 2026-05-27
**Start Time:** 14:08
**End Time:** 15:52
**Duration:** 104 minutes
**Problem:** Design a Web Crawler

---

## Conversation Log

**Interviewer:** Design a Web Crawler. The system should crawl the public web at scale, extract content from pages, store it for downstream consumers (search indexing, ML training corpora), and do so without overloading the sites it visits. Start by gathering requirements.

**Aayush:** the web crawler would need to extract contents of a webpage by parsing its HTML contents right?

**Interviewer:** Yes — clarified scope (raw HTML vs cleaned text, content types, link following).

**Aayush:** what would be the purpose of the web crawler

**Interviewer:** Assume search-engine indexing (Googlebot-like). Implications: broad coverage, freshness, politeness, downstream indexer.

**Aayush:** the index would be a keyword type index right?

**Interviewer:** Yes — inverted index downstream. But that's out of scope. Drive the requirements yourself: FRs, NFRs with numbers, out-of-scope.

**Aayush:**
FRs:
1. Parse HTML contents of a webpage to extract its text and linked URLs
2. Store extracted text corpus for downstream processing
3. Periodically refresh crawled content

Out of scope: non-HTML, media content

NFRs:
1. Durability via S3
2. 1-day freshness delay
3. Incrementally discover new URLs
4. 1B pages × 100KB = 1PB storage
5. 1000 pages/s throughput

**Interviewer:** Pushed back: storage math is wrong (10⁹ × 10⁵ = 10¹⁴ = 100TB, not 1PB). Throughput inconsistent — 1000/s × 86,400 = 86M/day vs 1B corpus → ~12 days per refresh cycle, not 1 day. Missing: availability SLA, politeness number, HTML vs extracted text.

**Aayush:** can you explain a bit about what the politeness constraint is?

**Interviewer:** Explained: respect robots.txt, per-domain rate limits, identify via User-Agent. Implications for URL frontier partitioning by domain.

**Aayush (revised):**
- Storage: 100TB
- Throughput: 10,000 pages/s
- Availability: 99.9% (later clarified — storage availability for corpus, not pipeline)
- Politeness: 1 req/s/domain default
- Page size: 100KB raw text, 1MB HTML → 10GB/s ingress

**Interviewer:** Math checks out. Now move to high-level design.

**Aayush:** SQL DB stores discovered URLs + yet-to-crawl. Process reads SQL → pushes to Kafka with topic per domain. Consumer groups poll and fetch. Custom domain rate limiter in front of fetches. Consumers parse HTML, update SQL with text, add new discovered URLs back to SQL.

**Interviewer:** Pushed back on three issues:
1. 100TB of text in SQL — wrong choice; honor your own S3 NFR
2. Topic per domain — Kafka caps at ~10K-100K topics, not 200M
3. URL dedup at scale

**Aayush:**
1. Store text in S3, store S3 URL in SQL
2. Use domain as Kafka partition key
3. Redis cache for seen URLs

Asked interviewer to draw the diagram.

**Interviewer:** Drew the diagram with the components Aayush described. Then pushed: Frontier Reader scalability, SQL write load (110K writes/s), and re-crawl scheduling.

**Aayush:** Move to DynamoDB for write throughput.

**Interviewer:** Probed: partition key, GSI on lastCrawlTime, cost, dedup with conditional writes. The harder question — re-crawl loop. Walk me through it.

**Aayush:** Frontier process queries DB at fixed intervals to find URLs with no lastCrawlTime or lastCrawlTime > 1d.

**Interviewer:** That hand-waves: scanning 1B rows requires index. GSI on lastCrawlTime is monotonically increasing → hot partition. Offered alternative: push model with delayed scheduling.

**Aayush:** Push model with delayed scheduling.

**Interviewer:** Made it concrete — pick an implementation: SQS delay (max 15 min), Kafka + scheduler, Redis sorted set, bucketed DynamoDB.

**Aayush:** Time-bucketed DynamoDB. Consumer inserts new record with next crawl time as partition key. Separate scheduler process queries current time bucket and pushes to Kafka.

**Interviewer:** Hot partition problem at fine granularity — need composite key `(time_bucket, shard_id)` with random shard. Also: exactly-once if scheduler crashes mid-batch.

**Aayush:** Yes, composite key with random shard_id in fixed range. Status=queued in DB when pushed to Kafka. Update the diagram.

**Interviewer:** Updated diagram with Scheduler DB + URL Metadata DB. Listed remaining deep-dives: Redis dedup cost, distributed rate limiter, fault tolerance, Bloom filter.

**Aayush:**
- B: Distributed rate limiter via Redis, N consumers speak to centralized interface.
- A: Bloom filter for URLs.
- C: Consumer crash — idempotent consumers, Kafka redelivers if offset not committed. Scheduler crash — fetch un-queued items from current bucket on restart. S3 write failure — fail the attempt and communicate. Kafka broker — replicas.

**Interviewer:** Pushed back:
- Partitioning by domain → single consumer per domain → local rate limiter is sufficient. Distributed rate limiter is over-engineering given the partitioning choice.
- Bloom filter false positives mean ~1% URLs never crawled (acceptable trade-off, document it).
- Multi-write idempotency across S3 + 2 Dynamo tables isn't automatic.
- S3 write failure: needs concrete fallback (retry, DLQ).

Asked for end time.

**Aayush:** 15:52.

**Interviewer:** Delivered full feedback (see Feedback section).

---

## Design Summary

**Requirements Gathered:**
- FRs: extract HTML text + links, store corpus, periodic refresh
- NFRs: 1B pages, 100TB storage, 10K pages/s, 1-day freshness, 1 req/s/domain politeness, 10GB/s ingress
- Storage availability: 99.9%

**High-Level Architecture:**
- Scheduler DynamoDB (PK = `(time_bucket, shard_id)`)
- Scheduler Process reads current bucket → Kafka (partitioned by domain)
- Crawler Consumers fetch via Per-Domain Rate Limiter → external Websites
- Extracted text → S3 blobs
- URL metadata (s3Url, lastCrawlTime, status) → URL Metadata DynamoDB
- Consumers insert next-crawl entry into Scheduler DB
- Redis-based seen-URL dedup

**Key Design Decisions & Trade-offs:**
- DynamoDB chosen over SQL for write throughput at 110K writes/sec
- S3 for text blobs, Dynamo for metadata only (honors durability NFR + reduces hot data cost)
- Kafka partition key = domain (natural per-domain ordering, simplifies rate limiting)
- Time-bucketed scheduler with random shard_id distributes hot writes
- Push-based delayed scheduling instead of pull-based polling

**Scalability & Fault Tolerance Points:**
- Sharded scheduler DB partitions avoid hot partitions
- Kafka replication for broker failover
- Idempotent consumer pattern with uncommitted-offset redelivery
- Scheduler restart by re-querying current time bucket

**Gaps / Missed Areas:**
- API design never formalized (no endpoints, request/response shapes)
- Did not initially see that domain-partitioning makes distributed rate limiter unnecessary
- Multi-write atomicity across S3 + 2 Dynamo tables not addressed concretely
- S3 write failure handling left vague
- Did not address content dedup (same content at multiple URLs)
- Did not draw the architecture diagram personally
- Did not volunteer break/fix in deep dives — required interviewer prompting
- Initial arithmetic slip (1PB vs 100TB)
- Asked the interviewer to define the use case

---

## Feedback Given

### Requirements Clarification — Mixed
- FRs clear, scope clear
- Needed multiple prompts for numbers, arithmetic slip on storage
- Throughput vs freshness inconsistency had to be flagged
- Politeness was hand-wavy until pushed for a number
- "Availability" confused crawler vs corpus availability
- Asked the interviewer for the use case rather than proposing one

### API Design — Not Covered
- No endpoints discussed, no request/response shapes

### High-Level Architecture — Reasonable
- Components in the right shape
- Good instinct on Kafka partitioning by domain

### Component Design & Trade-offs — Mixed
- Good pivots when pushed: SQL → Dynamo, large blob → S3
- Topic-per-domain (Kafka scale red flag) until corrected
- Full URL set in Redis until Bloom filter was suggested
- Did not see partition-by-domain → local rate limiter
- Component choices stated without reasoning

### Scalability & Fault Tolerance — Mixed
- Recognized hot partition after prompt; clean shard_id fix
- Kafka replication correct
- Consumer crash answer surface-level — didn't address multi-write idempotency
- S3 failure handling vague

### Deep Dive Quality — Mixed
- Time-bucketed scheduler design landed well
- Naive → break → fix was driven by interviewer, not Aayush
- Exactly-once re-scheduling only addressed when explicitly asked

### Communication — Below Bar
- Multiple prompts to articulate ideas
- Arithmetic slips (1PB vs 100TB)
- Asked interviewer to draw the architecture

### Diagram Quality — Below Bar
- Drew only requirements text in the diagram
- Did not draw any architecture components
- Interviewer drew the full architecture twice

### Top 3 Improvements
1. Drive the whiteboard yourself
2. Volunteer the naive → break → fix structure
3. Time budget — 104 min vs 45-60

**Time Taken: 104 minutes**
