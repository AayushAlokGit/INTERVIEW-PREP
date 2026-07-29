# System Design Round Transcript
**Date:** 2026-06-15
**Start Time:** 10:43
**End Time:** 11:43
**Duration:** 60 minutes
**Problem:** Design a Collaborative Document Editor (Google Docs)

---

## Conversation Log

**Interviewer:** Introduced the problem — multiple users edit the same doc, near real-time. Asked for requirements.

**Aayush:** Asked what scale to assume.

**Interviewer:** Redirected — propose your own assumptions and reason from them.

**Aayush:** FRs — (1) create docs, (2) share for editing, (3) multiple users edit + see changes real time, (4) see each user's cursor. Out of scope — versioning/undo, access-control layers. NFRs — p99 < 200ms edit propagation, strong consistency (linearizability), durability, 100M DAU, 10 concurrent editors/doc, 10k docs concurrently edited → 100k edits/s, fanout 900k edits/s.

**Interviewer:** Probed the consistency claim — linearizability vs convergence; tension with p99 200ms. Sanity-checked edits/s derivation; asked if edit = keystroke.

**Aayush:** Linearizability = everyone sees a write when it happens; edit = a keystroke (each char must be broadcast).

**Interviewer:** Pushed — strict linearizability means each keystroke waits for a central sequencer round trip (150ms+ transatlantic). Hold the line or commit to convergence?

**Aayush:** Agreed strict linearizability is problematic — user would wait for ordering/consolidation/re-render. Better: user sees own changes instantly, doc eventually converges. Committed to **eventual consistency**.

**Interviewer:** Locked requirements with that correction; rendered FRs + NFRs.

**Aayush:** (follow-up) Since eventual consistency, we can also have high availability — the system is **AP**.

**Interviewer:** Confirmed the CAP positioning (trade C for A under partition; partitioned editors keep editing locally and merge on reconnect). Flagged the conflict-merge problem as the deep dive. Asked for core entities.

**Aayush:** Document {id, userIds[], createdAt, updatedAt}; User {id, name}; DocumentEdit {userId, editText, editArea}; CursorPosition {userId, xCoordinate, yCoordinate}.

**Interviewer:** Probed — how does an edit identify *where* it applies (index shifts under concurrent inserts)? Where does the content live? Is pixel x/y the right cursor representation?

**Aayush:** Content in blob storage (S3), reconstructed from edits and flushed. Positioning must be render-agnostic but unsure of a method.

**Interviewer:** Confirmed logical char offset is the render-agnostic anchor; index-shift-under-concurrency is the deep dive; noted live state must also be held fast (not S3 per keystroke). Rendered entities. Asked for API design.

**Aayush:** Identity from JWT. POST /documents (req {name}, res Document, idempotency key). PATCH /documents/:id/share (req {userIds[]}). WebSocket for bidirectional. WS client→server: pushCursorPosition, pushDocumentEdit; server→client: receiveDocumentEdits, receiveCursorPositions.

**Interviewer:** Credited JWT, idempotency, WS justification. Asked for (1) field-level edit message shape incl. version/sequence, (2) bootstrap path for initial doc state.

**Aayush:** WS /documents/:id; on init server sends current doc state.

**Interviewer:** Bootstrap handled. Pressed again for the edit payload field shape + version.

**Aayush:** Not sure of the API details.

**Interviewer:** Supplied a baseline payload; deferred the version/position-under-concurrency field to the deep dive. Rendered API. Asked for HLD (write + read/bootstrap paths).

**Aayush:** Stateful API servers (maintain WS connections). API gateway in front = auth middleware + load balancing via consistent hashing on docId so co-editors land on the same node. Edit pushed via WS → server alters in-memory doc state, resolves concurrent edits to a consistent state, broadcasts to all users via WS. In-memory state meets latency budget. Periodically flush doc state to S3 (overwrite). Each edit also stored in DynamoDB (PK=docId, SK=createdAt) as an edit log.

**Interviewer:** Rendered HLD. Deep dive — made "resolve to consistent state" concrete: doc "HELLO"; A inserts X@0, B inserts Y@5 (append). Apply A → "XHELLO", then B@5 literally. What string, and is it what B wanted?

**Aayush:** Each user's edits applied to the version they're seeing, then resolved. (After being walked through indices) recognized XHELLYO ≠ intended XHELLOY.

**Interviewer:** Pinned the bug — A's insert shifted positions, B's position 5 is stale. What must the server do to B's edit before applying?

**Aayush:** Position must be incremented/decremented by a delta depending on the tail of edits applied before the current edit.

**Interviewer:** Named it — Operational Transformation. Asked (1) what the edit must carry so server knows the baseline/concurrent set, (2) name the alternative class of structures.

**Aayush:** Keep a monotonically increasing document version; sent to clients on broadcast, carried back on each edit to indicate the doc state the client saw; server uses it + edit history to compute the delta.

**Interviewer:** Confirmed (fills the deferred API field). Named the alternative (CRDT) and its trade-off. Then stressed fault tolerance: stateful node holds the only in-memory copy — it crashes. What's lost, how do editors recover?

**Aayush:** Current in-memory state lost, plus any un-persisted edits. Restore from S3 snapshot + replay DynamoDB edits with version > snapshot version → back to most recent persisted state.

**Interviewer:** Pinned the durability window — when is the edit written to Dynamo, before or after broadcast?

**Aayush:** Async write to DynamoDB to ensure no edits lost ever.

**Interviewer:** Corrected via crash-timeline trace — async creates a loss window; edits can be broadcast then lost. Which ordering guarantees no loss?

**Aayush:** Before broadcasting.

**Interviewer:** Correct. Noted the cost (per-keystroke synchronous write eats the latency budget). Asked for a cheaper durable append-only log.

**Aayush:** An append-only WAL like Kafka.

**Interviewer:** Confirmed — sequential append log gives durability-before-broadcast cheaply; materialize S3 snapshot + Dynamo log asynchronously. Wrapped.

---

## Design Summary

**Requirements Gathered:**
- FRs: create docs; share for editing; multi-user real-time editing with visible changes; per-user cursor.
- NFRs: p99 < 200ms propagation; eventual consistency / convergence (local-first edits), system is AP; durability of committed edits. Scale: 100M DAU, ~10 editors/doc, ~10k docs concurrently edited, ~100k edits/s ingest, ~900k broadcasts/s fanout.

**High-Level Architecture:**
- Clients (browsers) ↔ API Gateway (JWT auth + consistent-hash LB on docId) ↔ stateful API service nodes (hold WS connections + in-memory doc state, resolve concurrent edits via OT, broadcast).
- S3: periodic full-doc snapshot (overwrite).
- DynamoDB: edit log, PK=docId SK=createdAt.
- (Deep dive) Append-only WAL/Kafka as the fast durable log written before broadcast.

**Key Design Decisions & Trade-offs:**
- Eventual consistency over linearizability (reasoned: per-keystroke sequencer round trip = typing lag); AP under CAP.
- Operational Transformation for convergence (reasoned to it); document version on each edit as the OT baseline. CRDT named as the alternative (P2P/offline; trades per-char metadata + tombstones).
- Consistent hashing on docId to colocate co-editors on one stateful node.
- Durability: write to durable log BEFORE broadcast (corrected from async); WAL/Kafka to keep the hot path fast.

**Scalability & Fault Tolerance Points:**
- In-memory state for low latency; recovery = S3 snapshot + replay edits with version > snapshot version.
- Durable-write-before-broadcast guarantees no acknowledged edit lost.

**Gaps / Missed Areas:**
- Edit wire format not specified until prompted (stalled on the payload shape).
- Did not self-drive a scale-break: single stateful node per doc caps edit throughput + broadcast fanout; a single hot/viral doc doesn't shard (OT needs one serialization point). No mitigation (region-local relays, editor caps, batching) raised.
- Consistent-hash ring rebalancing: no plan for migrating in-memory state when a node joins/leaves.
- Operability: no monitoring of convergence/divergence, broadcast lag/backpressure at peak, or hot-doc detection.
- Cursor: didn't know logical-offset representation by name.
- Initial durability-semantics error (async "ensures no loss") — corrected with prompting.

---

## Feedback Given

### Standard Evaluation
- **Requirements — Strong.** Clean FR/NFR split, self-proposed scale numbers, and self-corrected linearizability → convergence by reasoning, then connected to CAP/AP unprompted (best sequence of the round).
- **Core entities — Good.** Sensible objects; correct instinct that cursor must be render-agnostic.
- **API — Mixed.** Strong envelope (JWT identity, idempotency key, WS w/ justification) but stalled on the load-bearing edit payload shape.
- **HLD — Strong.** Stateful nodes, consistent hashing on docId, in-memory state, S3 snapshots, Dynamo edit log — coherent.
- **Deep dive — Strong core / shaky durability.** Reasoned to OT unprompted; derived per-edit version baseline; correct recovery path. Durability ordering initially backwards (async), corrected cleanly; landed on WAL/Kafka.
- **Communication — Good.** Clear, self-corrects when traced through a scenario; still needs the scenario supplied.

### Senior-Signal Scorecard
| Signal | Rating | Reason |
|---|---|---|
| Owns the narrative / self-raises traps | Mixed | Self-raised CAP/AP; OT, durability, CRDT needed prompting |
| Leads with trade-offs vs named alternatives | Mixed | Strong on consistency/CAP; didn't name CRDT or justify Dynamo/S3 unprompted |
| Pushes scale until it breaks | Weak | No self-driven scale-break; never confronted single-node-per-doc cap |
| API as a designed contract | Mixed | Good envelope, stalled on edit payload shape |
| Operability / second-order concerns | Mixed | Good recovery when prompted; no monitoring/backpressure/rebalancing raised |
| Pace | Weak | 60 min vs 45–50; entities/API drifted |

**Overall:** Strong mid-level (L4) with senior flashes (CAP reasoning, reaching OT independently). Not yet senior strong-hire — sharp problems surfaced via interviewer steering, and pace ran over.

### What a senior strong-hire would have done
- Led with the OT-vs-CRDT fork and justified OT for a central-server model.
- Specified the edit wire format (incl. baseVersion) up front.
- Self-driven the scale-break: a single doc pins to one node (OT serialization point) → hot/viral doc caps CPU + fanout and doesn't shard; mitigations = editor caps, region-local relay/fanout servers, keystroke batching.
- Owned durability ordering immediately (WAL-before-broadcast + batching).
- Raised operability: divergence detection, ring-rebalancing state migration, broadcast lag at peak.

**Time Taken: 60 minutes**
