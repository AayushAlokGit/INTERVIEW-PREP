# System Design Round Transcript
**Date:** 2026-06-29
**Start Time:** 11:51
**End Time:** 12:32
**Duration:** 41 minutes
**Problem:** Design an Object Storage System (S3-like)
**Note:** Started as a scored mock; Aayush switched to TEACHING MODE after the API-design phase (unfamiliar with the system). Second half is collaborative teaching, not a graded interview.

---

## Conversation Log

**Interviewer:** Presented the problem — design an S3-like object storage system (buckets, object keys, durable, available, massive scale). Asked Aayush to drive requirements gathering.

**Aayush:** Asked for an example of blobs organized into buckets and who sets the organization.

**Interviewer:** Explained: bucket = user-created top-level container (globally unique name); object addressed by client-chosen key string (folder-style naming, no real dirs); identified by (bucket, key); client owns the organization.

**Aayush:** FRs: store & retrieve blobs; blobs in user-created buckets; unique key per blob (client-set, folder convention but no real dirs); (bucket,key) locates a blob. Out of scope: TTL. NFRs: highly durable (99.999), highly available (99.99 ~52min/yr), eventual consistency OK, infinite scalable storage.

**Interviewer:** Pushed: FR scope thin (delete? list? large objects?). NFRs are adjectives — need numbers. Asked for real durability target and derived scale.

**Aayush:** Added FRs: delete, list, upload (multipart/resumable for large uploads).

**Interviewer:** Pressed on numbers — durability target (S3 = 11 nines) and scale arithmetic.

**Aayush:** 1 trillion objects x 1 MB = 10^18 bytes. Read-heavy 100:1. Ingest: 1B DAU x 10 objects/day = ~10^5 objects/s x 1MB = 100 GB/s.

**Interviewer:** Confirmed math (1 EB, 100 GB/s). Asked for read QPS/throughput and what 11 nines forces.

**Aayush:** Read QPS 10^7/s. Durability -> replicate the object data.

**Interviewer:** Confirmed (read throughput ~10 TB/s; redundancy required). Locked requirements, rendered FRs/NFRs. Asked for core entities.

**Aayush:** Object(chunks:[], bucket, key, uploadedBy).

**Interviewer:** Pushed for more complete model — object fields (size, type, integrity, timestamps, versions), is Bucket an entity, what's in a chunk record, what tracks in-progress multipart.

**Aayush:** createdAt; Bucket as its own entity (name, createdBy); chunk record has the address of the chunk data; ObjectUploadStatus entity with chunksUploaded + status.

**Interviewer:** Noted size + checksum/integrity still missing; rendered his entities. Asked him to design the API (incl. multipart contract).

**Aayush:** Requested to switch to TEACHING MODE — still designing but wants to be taught (unfamiliar with the system).

**Interviewer (teaching from here):** Taught the core two-plane mental model (metadata plane vs data plane, opposite characteristics, scale independently). Rebuilt ObjectMetadata with size/contentType/etag-checksum/versionId and why each exists. Asked Aayush what versioning changes in each plane.

**Aayush:** The version of ObjectMetadata updates and the address points to new bytes.

**Interviewer:** Refined: a NEW metadata record per versionId (old row untouched -> old version still retrievable); new bytes go to NEW immutable chunks; old chunks linger until GC. Taught blob immutability as the unlocking principle. Taught the simple API (PUT/GET/DELETE/LIST with cursor pagination, ETag+versionId contract) and asked Aayush to sketch the 3-step multipart contract.

**Aayush:** Client chunks file, passes chunk details to server, server creates ObjectUploadStatus and returns uploading.

**Interviewer:** Completed the canonical 3-step multipart: (1) POST ?uploads -> uploadId; (2) PUT ?partNumber&uploadId -> ETag per part (parts immutable, independent, retryable = resumable); (3) POST complete with manifest -> atomic metadata stitch makes object GET-able; abort + GC for orphans. Rendered API. Taught HLD components (LB, stateless API/Storage Service, sharded Metadata Store, Data/Chunk Nodes, Placement, GC/Repair). Asked Aayush to trace the WRITE path.

**Aayush:** Client uploads bytes -> written to data plane -> metadata plane handed the link to the bytes, saved in DB.

**Interviewer:** Confirmed the key insight (bytes-first, metadata-last = durability decision; crash leaves GC-able orphans not dangling pointers). Taught replication (3x, 200% overhead) vs erasure coding (6+3, ~50%) trade-off. Asked Aayush to trace the READ path.

**Aayush:** Hit metadata plane first, then extract bytes from data plane, return to client.

**Interviewer:** Confirmed; added read-time checksum verification (catch/heal bit-rot) and parallel chunk reads. Set up hard problem 1 (metadata at 1e12 rows / 1e7 QPS — how to shard given LIST requirement).

**Aayush:** hash(bucket) + sort key makes LIST efficient. Shard-by-bucket fails on a hot bucket (one shard melts, other clients of bucket affected). To avoid hot key, shard by bucket+key but then LIST becomes scatter-gather across partitions.

**Interviewer:** Affirmed he identified both horns. Taught the resolution: range-partition sorted by (bucket,key) with DYNAMIC SPLITTING on hot/large shards (Bigtable/Dynamo style) -> cheap range-scan LIST + hotspot relief. Set up hard problem 2 (10 TB/s reads).

**Aayush:** (1) CDN to absorb load. (2) Hot/cold tiering.

**Interviewer:** Completed both: CDN absorbs the bulk, immutability makes caching trivially safe (version = cache key); hot/cold tiering (SSD+replication vs HDD+EC) for bandwidth + cost. Summarized the full design, rendered HLD + full optimal reference to the canvas.

---

## Design Summary
**Requirements Gathered:** FRs — upload (multipart/resumable), GET, delete, list-by-prefix. NFRs — 11 nines durability (corrected from his 5 nines), 99.99% availability, eventual consistency, 1 trillion objects x 1MB = 1 EB, 100:1 read:write, write 1e5/s, read 1e7/s, ingest 100 GB/s, read 10 TB/s.
**High-Level Architecture (taught/co-designed):** Client -> CDN (reads) / LB (writes) -> stateless API/Storage Service -> Metadata Store (range-partitioned, sorted by bucket,key, dynamic split) + Data/Chunk Nodes (immutable chunks, replication or EC across AZs, per-chunk checksum) + Placement/Allocator; GC + Repair/Scrub worker in background.
**Key Design Decisions & Trade-offs:** two-plane separation; immutable versioned blobs; bytes-first/metadata-last commit; replication vs erasure coding; range-partition vs hash sharding; CDN + immutability for cache safety; hot/cold tiering.
**Scalability & Fault Tolerance Points:** redundancy across failure domains for 11 nines; read-time checksum + scrub + auto re-replication; dynamic shard split for hot/large buckets; CDN for read scale.
**Gaps / Missed Areas (before teaching mode):** independent scale numbers and durability target needed prompting; entities thin (missed size, checksum/etag, contentType, versionId until taught); did not reach API/HLD/deep-dives independently — these were taught.

---

## Feedback Given (learning-oriented — not a graded mock)

**Where Aayush did genuinely well**
- **Requirements numbers, once pushed, were derived cleanly and correctly** — 1 EB total, 100 GB/s ingest, 10^7 read QPS all computed without arithmetic slips.
- **Excellent reasoning on the metadata sharding dilemma** — independently identified BOTH the hot-bucket failure of shard-by-bucket AND the scatter-gather cost of shard-by-(bucket+key). Naming both horns of that trade-off is a senior-level instinct; he just hadn't seen the range-split resolution.
- **Nailed the write-path ordering** (bytes first, then metadata pointer) and the read-path order — the load-bearing correctness insights.
- **Good instincts on the read-scale problem** (CDN + hot/cold tiering) even while unfamiliar.

**What to drill (this was new material, so these are learning targets, not failures)**
- **Lead requirements with numbers and the right durability bar.** Object storage = 11 nines, not 5; that single number drives the entire redundancy design. State avg+peak numbers unprompted.
- **Model entities completely.** size, checksum/etag (integrity), contentType, versionId are load-bearing — integrity especially, since it is how 11-nines durability is actually delivered.
- **The two-plane model (metadata vs data) is the reusable spine** — internalize it; it recurs in any blob/file/media system.
- **Immutability as a design lever** — immutable versioned chunks make replication, CDN caching, and integrity all simpler. Reach for it.

**Concepts taught this session (study list):**
two-plane architecture; immutable versioned blobs + version-as-new-record; 3-step multipart/resumable contract; bytes-first/metadata-last commit ordering; replication vs erasure coding; per-chunk checksum + read-time verification + scrub; range-partitioned metadata with dynamic splitting; CDN + immutability for read scale; hot/cold tiering; GC of orphan chunks.

**Time Taken: 41 minutes** (teaching session; ~first third was scored-style requirements/entities, remainder collaborative teaching).
