# System Design Round Transcript
**Date:** 2026-05-22
**Start Time:** 12:30
**End Time:** 13:54
**Duration:** 84 minutes
**Problem:** Design a Search Autocomplete / Typeahead system

---

## Conversation Log

**Interviewer:** Design a Search Autocomplete / Typeahead system. As a user types characters, the system suggests the most relevant completions in real time (e.g. "new y" -> "new york", "new york times", "new year"). Start by gathering functional and non-functional requirements, with numbers on the NFRs.

**Aayush:** Do we only need prefix matching for the returned suggestions?

**Interviewer:** Yes — prefix matching is the core requirement. Fuzzy/typo tolerance is out of scope. Decide explicitly: how many suggestions (top-K), what "most relevant" means (ranking), and where the candidate phrases come from.

**Aayush:** Let's say returning 5 suggestions. We could rank the suggestions based on hit count for each suggestion or popularity.

**Interviewer:** Good. Where do the phrases come from — derived from past user searches, or a curated corpus? And how fast must a newly trending term show up (freshness)? Now give me the NFRs with numbers — scale, latency, availability, read:write ratio.

**Aayush:** Can you explain the freshness question a bit more?

**Interviewer:** [Explained freshness — the lag between a phrase becoming popular and it appearing as a suggestion; spectrum from real-time (seconds) to near-real-time (minutes) to batch (hours/daily); explained that the choice is a major architectural fork between a live-updating structure and an offline batch rebuild.]

**Aayush:** FRs: (1) suggest relevant completions for a prefix string, (2) return top 5 ranked by popularity, (3) add new phrases when encountered from user search queries. Out of scope: fuzzy matching.

**Interviewer:** FRs are crisp. But you skipped the phrase source, freshness, and NFRs. Give me NFRs with numbers — DAU, QPS (a request fires per keystroke), latency + percentile, availability, read:write ratio.

**Aayush:** Can you explain the freshness question a bit more? [clarification provided]

**Aayush:** NFRs: (1) highly available (99.9% ~ 8.6 hrs downtime/yr), (2) low latency (p99 < 100ms), (3) eventual consistency acceptable for popularity, (4) 10M DAU, 5 searches/day, 100 chars per search (~100 calls per search) -> ~50k QPS for autocomplete, (5) delay between a phrase crossing the popularity threshold and becoming searchable should be < 5s.

**Interviewer:** Good — percentiles and quantified availability. Three things: check the arithmetic (5B calls/day / 86,400s); does every keystroke really fire a call (debouncing)?; and <5s freshness is effectively real-time — is that intentional?

**Aayush:** I'm assuming 10^5 seconds in a day for ease of calculation, and peak will be 10x.

**Interviewer:** That's a legitimate BoE shortcut — ~50k avg, ~500k peak. That 500k drives the architecture. Now move to high-level design.

**Aayush:** Before moving to high-level design I want to get the core entities and APIs down. Core Entities: (1) Phrase, (2) Suggestion. APIs (userId from JWT): (1) GET /suggestions?query={} -> Suggestion[], top 5 by popularity. (2) POST /phrases -> Phrase, create a new phrase, fired when no suggestions returned (i.e. new phrase). Body: {phraseText}.

**Interviewer:** You led with auth — good. But the entities have no fields, the response shape is undefined, and the write path is under-modeled: ranking is by popularity, but POST /phrases only fires when zero suggestions return, so already-popular phrases never get their counts incremented. Also POST /phrases lets any client inject arbitrary text — abuse. Fix the write path so popularity is captured.

**Aayush:** To prevent abuse we'd need a mechanism to determine if a newly seen phrase is garbage — that adds complexity to the design.

**Interviewer:** You may not need a garbage classifier. Your freshness NFR uses a popularity threshold — a phrase one spammer types never crosses it. The threshold IS the abuse filter. So drop client-submitted phrases as suggestions until they earn their way in by being searched. What event does the client send, and when?

**Aayush:** [Simplified] Core Entities: Phrase (text, popularity_count, updatedAt, createdAt). API: PUT /suggestions?query={} -> PhraseText[]. Gets top 5 by popularity AND adds the phrase / increments its popularity count.

**Interviewer:** That's an anti-pattern. (1) Wrong verb — a per-keystroke read must be GET (safe, idempotent, cacheable). (2) You merged read and write — typing "new york" would increment popularity for "n", "ne", "new"... partial prefixes counted as searches; and 500k reads/s become 500k writes/s. A keystroke is not a search. You need two endpoints: GET /suggestions (per keystroke, ~500k QPS) and POST /searches (committed search, ~50k QPS).

**Aayush:** Clarifying — on every keystroke we get suggestions, and we put in a new phrase only when the user presses enter, right?

**Interviewer:** Exactly. Every keystroke -> GET /suggestions (read only). On enter/click -> one POST /searches: upsert + increment popularity (create if new). Now go to high-level design.

**Aayush:** The data structure should be a prefix trie. Nodes contain a list of child nodes (26, one per letter, single case) and a count = popularity count of the word ending at that node. The trie must be maintained entirely in memory.

**Interviewer:** Trie is right. But walk me through retrieving the top-5 for prefix "n" — the actual phrases end deep below the "n" node. How do you get top-5 by popularity, within p99 < 100ms at 500k QPS?

**Aayush:** We could store the top 5 most popular suggestions following the char at each node in a max-heap, and when adding a new phrase, adjust the heaps of the parent nodes while recursing back up.

**Interviewer:** That's the key optimization — precompute and cache top-5 at every node; lookup becomes O(prefix length). (Minor: use a min-heap of size 5, or just a sorted array.) Now — sizing, serving 500k QPS, and concurrency: writes mutating the trie while 500k reads/s traverse it.

**Aayush:** Sizing: ~1B distinct phrases, ~100 chars each -> ~1 TB of data. The trie needs even more memory for the popularity counts.

**Interviewer:** Arithmetic: 1B × 100 bytes = ~100 GB, not 1 TB. But you're right the trie is much bigger — the 26-pointer child array per node plus per-node heap makes it several hundred GB to ~1 TB. That doesn't fit one box. So: (1) how do you shard the trie? (2) how do you handle writes vs 500k concurrent reads? (<5 min freshness gives you room.)

**Aayush:** Shard by initial letter of query -> 26 shards, each a trie rooted at one letter. Use leader-follower replication: writes go to the leader, reads to followers — eventual consistency, with delay before popularity counts show up.

**Interviewer:** Both have edges. First-letter sharding gives 26 wildly unequal shards (skew), and 26 is a hard ceiling — if the "s" shard still doesn't fit, you've solved nothing. And for replication: what actually travels leader -> follower, and how does a follower install an update without readers seeing a half-updated trie?

**Aayush:** (1) Use a hashing function to generate a hash for the prefix; with truly random hashing, use consistent hashing to distribute the trie data. (2) Replication: pass a write-ahead log to followers; followers periodically rebuild their local trie to incorporate the changes.

**Interviewer:** #2 works — make explicit that the follower builds a NEW trie and atomically swaps the root pointer, so readers never see a torn trie. #1 has a bug: hashing the prefix puts "n" and its child "ne" on different shards — you've cut the parent-child edges of your own tree. Either keep an entire subtree on one shard, or serve from a flat prefix->top-5 KV map.

**Aayush:** Let's partition by including the first 2 characters of the prefix.

**Interviewer:** Reasonable — ~676 partitions, a hot one can split to 3 chars. Two gaps: prefixes shorter than 2 chars (the routing key needs 2), and you need a component that routes a query to the shard owning its prefix.

**Aayush:** (1) We can suggest only after 2 chars — it simplifies things. (2) There needs to be a load balancer sitting in front of the trie nodes.

**Interviewer:** You're conflating two jobs. A load balancer spreads traffic across interchangeable replicas. You also need a routing layer (query router / gateway) holding the partition map — content-based routing on the prefix. Read path: client -> router -> shard load balancer -> follower replica -> trie lookup.

**Aayush:** [Asked the interviewer to update the draw.io file with the discussed architecture.]

**Interviewer:** [Updated the draw.io file with the full read path, write path, edges, and a design notes box.]

**Interviewer:** Please give me your current time so I can compute duration, then I'll deliver feedback.

**Aayush:** 13:54.

**Interviewer:** [Delivered full feedback — see below.]

---

## Design Summary

**Requirements Gathered:**
- FRs: prefix-match suggestions; return top 5 ranked by popularity; harvest new phrases from committed user searches.
- Out of scope: fuzzy/typo matching.
- NFRs: 99.9% availability; p99 < 100ms latency; eventual consistency acceptable for popularity; 10M DAU, ~50k QPS avg / ~500k QPS peak; freshness initially <5s, relaxed to <5 min.

**High-Level Architecture:**
- Read path: Client (keystroke) -> CDN/Edge Cache -> API Gateway -> Query Router (partition map) -> Shard Load Balancer -> Suggestion Service / Follower Trie Replicas.
- Write path: Client (enter) -> Write/Ingestion Service -> Popularity Counter Store -> Leader Trie -> WAL -> replicate to followers -> periodic rebuild -> atomic root-pointer swap.
- Data structure: in-memory prefix trie; each node caches its top-5 phrases (heap of size 5) so a lookup is O(prefix length).

**Key Design Decisions & Trade-offs:**
- Separate GET (read, per keystroke) from POST /searches (write, per committed search) — avoids corrupting popularity with partial prefixes and avoids turning reads into writes.
- Precompute top-5 at every trie node — trades extra write-time work and memory for O(prefix length) reads.
- Shard by first 2 chars of prefix (~676 partitions), keeping each subtree intact on one shard so traversal is local; hot shard split to 3 chars.
- Leader-follower replication via WAL; followers serve immutable snapshots and atomically swap the root pointer on rebuild — eventual consistency, acceptable per NFR.
- Popularity threshold doubles as the abuse/spam filter — no separate garbage classifier for the basic case.
- Only autocomplete from prefix length >= 2 — sidesteps cross-shard fan-out for short prefixes.

**Scalability & Fault Tolerance Points:**
- Peak 500k QPS handled via read replicas behind a per-shard load balancer.
- Sharding for both data size and load distribution.
- Concurrency solved with immutable snapshots + atomic pointer swap.
- CDN/edge caching of GET responses to absorb read load.

**Gaps / Missed Areas:**
- Arithmetic slip: 1B × 100 bytes = 100 GB, stated as 1 TB.
- Read:write ratio never stated as an explicit number.
- API response shape never fully specified (no explicit fields/score).
- Two HTTP verb mistakes (PUT for a read; PUT for a collection create).
- Initial endpoint merged read and write (regression introduced while "simplifying").
- Fault tolerance barely covered: no leader failover, no follower-failure handling, no replication factor, no circuit breakers, no graceful degradation; 99.9% NFR never tied back to design.
- Did not self-drive any naive->break->fix cycle — every "break" was supplied by the interviewer.
- Idempotency on the write endpoint not addressed.
- Diagram lagged the verbal design throughout; architecture was largely drawn by the interviewer on request.
- API box in the diagram left stale (contradicted corrected verbal design).

---

## Feedback Given

**Problem:** Design a Search Autocomplete / Typeahead system
**Time Taken: 84 minutes**

### Requirements Clarification — 6/10
- Good: FRs crisp; fuzzy matching scoped out explicitly. Final NFRs strong — p99 latency with percentile, availability quantified, peak QPS derived. The 10^5 s/day BoE shortcut was legitimate.
- Weak: NFRs emerged only after 3+ prompts; went straight to entities/APIs. Arithmetic slip: 1B × 100 bytes = 100 GB, said 1 TB (10x). Read:write ratio never stated numerically.

### High-Level Architecture — 5/10
- Final architecture is sound but Aayush did not draw it — drew 3 boxes and asked the interviewer to complete the diagram. Verbal design ran far ahead of the drawn design the whole round.

### API Design — 4/10
- Verb mistakes twice: PUT for a per-keystroke read (not safe/idempotent/cacheable — kills CDN); PUT /phrases for a collection create (should be POST).
- Merged read and write into one endpoint — would corrupt the ranking signal with partial prefixes and turn 500k reads/s into 500k writes/s.
- Response shape stayed vague (PhraseText[], no explicit fields/score).
- Good: led with JWT auth — a genuine improvement over prior rounds.

### Component Design & Trade-offs — 6.5/10
- Trie was right; top-5-cached-at-each-node is the crux and he got it — but only after the interviewer set up the "subtree explosion" break.
- Sharding first-letter -> 2-char was a good correction, again after the skew problem was flagged.
- Leader/follower + WAL — solid instincts; the atomic root-pointer swap had to be supplied.
- Rarely states reasoning for a choice before being asked.

### Scalability & Fault Tolerance — 5/10
- Scale handled (peak QPS, sharding, read replicas).
- Fault tolerance barely touched: no leader failover, follower failure, replication factor, circuit breakers, or graceful degradation; 99.9% NFR never tied back to design.

### Deep Dive Quality — 5.5/10
- Fixes well once a problem is on the table.
- Never volunteered a break — every naive->break->fix cycle had the interviewer supply the break.

### Communication & Process — 5/10
- Good early clarifying question (prefix-only matching).
- Needed repeated prompts to articulate ideas; over-simplified into a regression (merged endpoints).
- Time: 84 minutes — far over a real 45–60 min round; ~50 min spent before drawing a single box.

### Diagram Quality
- Final diagram reflects the design but was largely built by the interviewer on request; Aayush's own contribution was 3 boxes.
- API box left stale — still shows PUT /phrases and an unqualified "fired on every keystroke" GET, contradicting the corrected verbal design.
- Data flow clear and directional once drawn; key components present in the final version.

### Top 3 Things to Fix
1. Draw as you talk — the diagram must keep pace with your words, and never outsource it.
2. Self-drive the deep dive — say "the naive version breaks here, my fix is X" without waiting for the interviewer.
3. HTTP verbs (GET = safe/cacheable reads, POST = collection creates) and lead with reasoning before being asked.

**Strongest moment:** connecting the popularity threshold to abuse prevention — "the threshold is the spam filter."
