# System Design Round Transcript
**Date:** 2026-06-16
**Start Time:** 11:38
**End Time:** 12:31
**Duration:** 53 minutes
**Problem:** Design a Distributed Email Service (Gmail-like)

---

## Conversation Log

**Interviewer:** Introduced the problem — design a Gmail-like email service (send, receive, read inbox, search across years of mail). Asked for requirements first.

**Aayush:** What scale should the system support?

**Interviewer:** Redirected — proposing scale is part of the evaluation. Anchored at ~1B total users, ~200M DAU; asked him to derive emails/day, QPS (avg + peak), storage growth, and to give FRs first.

**Aayush:**
FRs — (1) send email to other users, (2) users have own inbox, (3) receive emails, (4) search through mails. Out of scope: spam filtering.
NFRs — (1) HA 99.99% (~52 min/yr), eventual consistency OK for mail; (2) low-latency transfer p99 < 200ms; (3) 1B users, 200M DAU, 5 sends/day, 3 reads/day x 20 emails; ~10k sends/s, ~40k reads/s at peak (5x); (4) low-latency search p99 < 500ms.

**Interviewer:** Flagged that send and read numbers weren't from the same baseline (sends ~11.5k/s avg, reads ~7k/s avg). Asked to restate avg vs peak consistently, and pushed for the skipped storage estimate.

**Aayush:** (1) avg sends 11.5k/s, avg reads 7k/s. (2) Assuming 100KB/email, ~100 TB email data/day.

**Interviewer:** Confirmed 100 TB/day → ~36 PB/yr (the dominating number). Locked requirements, rendered FRs + NFRs. Asked for core entities.

**Aayush:** Core entities — User(id, name, email); Email(id, from, text, toList, ccList, bccList); Inbox(id, userId). Simple version, will evolve.

**Interviewer:** Probed: how does Inbox connect to Email? When A sends to B and C, what gets created — one shared Email or a copy per recipient? Also flagged a missing field needed for ordering/search.

**Aayush:** One Email exists, referenced by InboxEmailRecord (link to inbox + email). One Email, many InboxEmailRecords. InboxEmailRecord has createdAt used to sort.

**Interviewer:** Accepted the shared-Email + per-recipient-pointer model. Rendered entities. Asked for the API — verbs, paths, request/response shapes, pagination, retry behavior.

**Aayush:** Identity from JWT. (1) POST /emails, req {toList, ccList, bccList, emailBody}, res 2xx, Idempotency-Key header. (2) GET /emails?cursor=&limit=&query=, res Email(from, toList, ccList, emailBody, createdAt)[], cursor pagination with cursor = createdAt + emailId.

**Interviewer:** Credited idempotency key + cursor pagination. Flagged: no single-email GET; list returns full body (heavy); response lacks an id. Asked for corrections.

**Aayush:** Add a single-email GET endpoint; emailBody only in single-email GET, not in list; id returned in list; list response also has nextCursor.

**Interviewer:** Rendered API. Asked for HLD — components, write path (A sends → lands in recipient inboxes), read path (B reads inbox), datastores named concretely.

**Aayush:** (scoped out notifications + attachments). Write path: client POST → API gateway (auth) → mail service → create Email record + InboxEmailRecords for to/cc/bcc. Body small, so store inline in primary DB (Postgres) rather than separate blob store. Postgres for indices + ACID transaction (all InboxEmailRecords in one txn so no mail lost). Inverted indices on body/subject for keyword search. Horizontally scale stateless mail service behind a load balancer. At this throughput SQL melts → switch to Cassandra, partition by userId so a user's InboxEmailRecords co-locate; writes go to multiple user partitions. Cassandra lacks transactions, so to ensure durability introduce a Kafka queue + idempotent workers. Mail service enqueues a message on new email; worker consumes, idempotently writes the Email record, then enqueues a message per recipient; downstream workers idempotently create InboxEmailRecords. Kafka holding each per-recipient message guarantees delivery durability. DLQ for failure investigation.

**Interviewer:** Credited the self-driven scale-break narrative. Rendered HLD faithfully. Probed: (1) search — inverted index made sense on Postgres but the store is now Cassandra (no full-text); how does search actually work, what owns the index? (2) partition by userId — what does a long-lived heavy user's single partition look like; when does it break and what do you do?

**Aayush:** (1) CDC-fed Elasticsearch instance for keyword search to narrow email scope. (2) Partition fills with InboxEmailRecords; split the partition across multiple nodes to distribute writes, but reads then query multiple partitions and merge. With 1-year retention it may not be an issue.

**Interviewer:** Credited CDC→ES and the scatter-gather observation. Sharpened: (A) inbox reads are always newest-first by createdAt — is there a partition scheme that avoids hitting every partition? (B) idempotency mechanism — exactly how does a worker avoid duplicate InboxEmailRecord under at-least-once? (C) operability — async pipeline, how do you know delivery is falling behind?

**Aayush:** (1) userId + timeBucket so recent emails land in same partition and inbox served from one partition. (2) Check DB for an existing InboxEmailRecord for the email+user. (3) Monitor createdAt of InboxEmailRecord vs createdAt of Email to measure how long the message sat in the queue per recipient.

**Interviewer:** Confirmed 1 and 3. Pushed on 2 — read-before-write adds a read at 58k/s peak and has a race (two redelivered copies both read "not found" and both write). Asked for idempotency by construction via the primary key.

**Aayush:** Enforce uniqueness on emailId + userId of InboxEmailRecord — a second worker writing the same thing fails (in Cassandra, a same-key write is a harmless upsert).

**Interviewer:** Confirmed — `(userId, emailId)` primary key makes the write naturally idempotent, no read, no race. Wrapped up and moved to feedback.

---

## Design Summary
**Requirements Gathered:**
- FRs: send email, per-user inbox, receive into inbox, search inbox. Out of scope: spam filtering (and later notifications, attachments).
- NFRs: HA 99.99%, eventual consistency OK; transfer p99 < 200ms; search p99 < 500ms; 1B users / 200M DAU; ~11.5k sends/s avg (~58k peak), ~7k reads/s avg (~35k peak); 100KB/email → 100 TB/day → ~36 PB/yr.

**High-Level Architecture:**
- Client → API Gateway (auth) → Load Balancer → stateless, horizontally-scaled Mail Service.
- Mail Service enqueues to Kafka; idempotent workers write the Email record and fan out per-recipient messages back to Kafka; downstream workers write InboxEmailRecords. DLQ for failures.
- Store: Cassandra (Email + inline emailBody; InboxEmailRecords partitioned by userId, later userId + timeBucket). Idempotency by `(userId, emailId)` primary key.
- Search: CDC-fed Elasticsearch on subject/body.

**Key Design Decisions & Trade-offs:**
- Shared Email + per-recipient InboxEmailRecord pointer (saves storage vs full copy per recipient).
- Inline email body in DB vs separate blob store — chose inline to avoid complexity (flagged in feedback as worth reopening at 36 PB scale).
- Postgres → Cassandra for write throughput; accepted loss of multi-row transactions, recovered durability via Kafka + idempotent workers + DLQ.
- userId + timeBucket partitioning to bound partition size and keep newest-first reads cheap.

**Scalability & Fault Tolerance Points:** horizontal stateless scaling behind LB; Kafka-buffered async fanout for durability; DLQ for poison messages; idempotent-by-key writes; time-bucketed partitions; delivery-lag monitoring via createdAt deltas.

**Gaps / Missed Areas:**
- Did not self-raise fanout amplification for large distribution lists / mailing aliases.
- Inline 100KB bodies in Cassandra at 36 PB/yr — object storage for bodies + metadata in Cassandra is the standard; not reconsidered.
- Storage tiering / hot-cold archival left implicit.
- External SMTP interop (MX, external delivery retries) not scoped.
- Asked interviewer for scale instead of proposing; storage skipped until prompted; avg/peak initially inconsistent.

---

## Feedback Given

### Requirements Clarification — 4/5
Clean FRs with explicit out-of-scope. Derived numbers but asked for scale first instead of proposing it, skipped storage until demanded, and mixed avg/peak until reconciled. Math sound once anchored (100 TB/day → 36 PB/yr).

### Core Entities — 4.5/5
Landed the central model correctly when probed: shared Email + per-recipient InboxEmailRecord pointer + createdAt. Minor nudge needed for timestamp and join entity.

### API Design — 4.5/5
Strongest API showing in a while. Unprompted idempotency key + tie-broken cursor pagination. Minor fixes: single-email GET, id/nextCursor in list response.

### High-Level Architecture — 5/5
Self-driven senior narrative: pushed Postgres until it broke, switched to Cassandra, caught lost atomicity, recovered durability with Kafka + idempotent workers + DLQ.

### Component Design & Trade-offs — 4.5/5
Named alternatives alongside choices (Postgres vs Cassandra; inline vs blob). Inline-body trade-off worth reopening at scale.

### Scalability & Fault Tolerance — 4.5/5
userId + timeBucket bucketing; DLQ; idempotency-by-key after a push off read-before-write.

### Deep Dive Quality — 4.5/5
CDC→Elasticsearch crisp and correct; partition-break reasoning good; idempotency took one push; delivery-lag monitoring a strong operability answer.

### Communication — 4.5/5
Tight, led the narrative, self-corrected. 53 min, slightly over target but improved.

**Overall: ~4.5/5 — best system design round in this set.**

---

## Senior Readiness Debrief

### Senior-Signal Scorecard
| Signal | Rating | Why |
|---|---|---|
| Own the narrative / self-raise traps | Strong | Self-raised durability loss, idempotency, DLQ unprompted. |
| Lead with trade-offs vs named alternatives | Strong | Postgres vs Cassandra, inline vs blob, justified. |
| Push scale until it breaks | Strong | Drove the Postgres→Cassandra break himself; engaged partition break. Historically weakest — big jump. |
| API as a designed contract | Mixed | Idempotency + cursor unprompted; single-GET and response id needed prompting. |
| Operability / second-order concerns | Mixed | Excellent lag-monitoring answer, but only when asked; didn't self-raise fanout. |
| Pace | Mixed | 53 min, close to target and much improved; drifted slightly in requirements. |

**Level read: senior, leaning strong-hire on this problem.** Remaining gap: self-raising the last 20% (operability, fanout trap) without prompting.

### What a senior strong-hire would have done on THIS problem
1. Self-raised fanout amplification — one send to a large distribution list = 1 write → N InboxEmailRecords (write storm); name it and consider mailing-list expansion handling.
2. Reopened "body inline in Cassandra" at 36 PB/yr — standard is metadata in Cassandra + bodies in object storage (S3/blob); inline bloats the partitions he's fighting to keep small and blocks tiering.
3. Made storage tiering / retention explicit — recent hot, older archived cold; also rescues partition size.
4. Scoped external SMTP interop (MX lookup, external delivery retries) even if out of scope.

### Self-drill
Review system_design_senior_guidance.md §1 (self-raise last traps: fanout, tiering) and §4 (name every read endpoint up front). At senior bar on scale and trade-offs; delta is surfacing operability/second-order concerns without prompting.

**Time Taken: 53 minutes**
