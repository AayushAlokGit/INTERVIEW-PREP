# System Design Round Transcript
**Date:** 2026-05-29
**Start Time:** 15:15
**End Time:** 16:11
**Duration:** 56 minutes
**Problem:** Design a File Storage & Sync System (Dropbox-like)

---

## Conversation Log

**Interviewer:** Presented the problem with facts: files up to 5 GB, avg 1 MB; 100 M users, 10 M DAU; 200 files/user; 10:1 R:W. Asked for FRs and NFRs.

**Aayush:** "should we support large files?"

**Interviewer:** Confirmed up to 5 GB, shared the rest of the facts.

**Aayush:** "add these facts to the drawio file"

**Interviewer:** Added Given Facts text block.

**Interviewer:** Pushed for FRs / NFRs.

**Aayush:** "can we consider file versioning out of scope?"

**Interviewer:** Yes. Stated scope explicitly — in: upload/download/sync/browse/delete; out: versioning, sharing, collab editing, search, push.

**Aayush:** "lets add the FRs to the drawio"

**Interviewer:** Added In Scope / Out of Scope blocks.

**Aayush:** NFRs v1 — resumable downloads, sync <1min, ~20PB storage, 1k reads/s avg, 99.9% upload availability.

**Interviewer:** Pushed for resumable uploads (not just downloads), durability target, consistency model, peak vs avg, latency SLA.

**Aayush:** NFRs v2 — added resumable uploads, eventual consistency, peak 10x = 10k reads/s, "use S3" for durability, latency targets (list folder p99 < 500ms).

**Interviewer:** Flagged "use S3" is implementation, not NFR. Should state target (11 nines) explicitly.

**Aayush:** "Highly durable file storage - s3" — kept implementation-leaning phrasing.

**Interviewer:** Accepted, moved on. Asked for high-level architecture.

**Aayush:** Listed entities (File, FileMetadata, Folder, User, Client).

**Interviewer:** Pushed back — that's a data model, not architecture. Asked for services, upload flow, download flow, sync mechanism.

**Aayush:** "add the data model to the drawio file"

**Interviewer:** Added data model entity boxes. Re-asked for architecture.

**Aayush:** Upload path — Client App → API Gateway → FileService. FileService generates presigned S3 URL, returns to client. Client uploads directly to S3 via presigned URL. Avoids proxying bytes through servers.

**Interviewer:** Pushed on 5GB resumability and metadata consistency.

**Aayush:** Multipart upload — save session details in DB, client resumes from session on crash. S3 event fires when upload complete → updates metadata status. Avoids client-side acknowledgment (security).

**Interviewer:** Pushed on dropped/delayed S3 events.

**Aayush:** Background reconciliation job scans `pending` files, queries S3 multipart status.

**Aayush:** "add these as diagrams to the drawio file"

**Interviewer:** Added Upload Path diagram with Client/API GW/FileService/DB/S3/Reconciliation Job + labeled edges.

**Aayush:** Download path — client runs recurring poll (45s) calling GET endpoint with last sync timestamp; server returns list of changed fileIds; client queries FileService for presigned download URLs. SSE alternative discussed for near-realtime, with tradeoff acknowledged (stateful service complexity).

**Interviewer:** Pushed on polling load (10M × 1/45s = ~220K req/s, 22x peak NFR) and on the supporting index for "files changed since T."

**Aayush:** Acknowledged load problem, FileMetadata has updatedAt + userId. For most users updates are rare, so SSE is better.

**Interviewer:** Asked about cross-server SSE fanout — file updated on server S1, user's other devices connected to S5/S12, how does the notification propagate?

**Aayush:** Proposed hybrid — polling on longer intervals + SSE. New FileSyncService keeps TCP alive, streams SSE on file updates, subscribes to Redis channel keyed by user. FileService publishes to user's Redis channel on update; FileSyncService fans out via active connections.

**Interviewer:** Probed reconnection scenario — Redis pub/sub is fire-and-forget; events between disconnect and reconnect are lost.

**Aayush:** "the regular polling on larger intervals will catch this."

**Interviewer:** Confirmed. Suggested immediate poll on reconnect as faster recovery than waiting for interval.

**Aayush:** "update the drawio now with the download path as well"

**Interviewer:** Added Download & Sync Path diagram — Client/API GW/FileSyncService/DB/Redis Pub/Sub/FileService/S3 with all labeled edges.

**Interviewer:** Last probe — file deletion sync.

**Aayush:** Soft delete via `isDeleted` column. Polling picks up the row because updatedAt bumps. Client deletes locally on seeing isDeleted=true.

**Interviewer:** Confirmed. Wrapped up.

---

## Design Summary

**Requirements Gathered:**
- FRs: upload (resumable, up to 5GB), download (resumable), sync <1min, browse, soft delete
- NFRs: 20PB storage, 1k reads/s avg / 10k peak, 99.9% upload availability, 11-nines durability (via S3), eventual consistency for reads, list folder p99 <500ms, sync interval <1min

**High-Level Architecture:**
- Client App with API Gateway entry
- FileService (presigned URL generation for upload + download)
- FileSyncService (long-poll + SSE, subscribes to Redis)
- FileMetadata DB (SQL) with updatedAt + userId indexed for change-feed queries
- Object storage: S3 with multipart upload for resumability
- Redis Pub/Sub (per-user channels) for cross-server SSE fanout
- Reconciliation Job (background) for missed S3 events

**Key Design Decisions & Trade-offs:**
- Presigned URLs for direct client-S3 transfer (avoids proxying bytes, scales independently of service compute)
- Multipart upload + DB-tracked session for resumability over flaky networks
- S3 event → metadata flag (server-side, not client-attested) to prevent malicious early-completion
- Reconciliation job as safety net for dropped events
- Hybrid sync: SSE for low-latency + long-interval polling for reliability and missed-event recovery
- Soft delete with isDeleted flag — sync flows through normal updated-at change feed

**Scalability & Fault Tolerance Points:**
- Direct S3 transfer keeps service compute decoupled from file bandwidth
- Multipart + reconciliation handles upload failures
- SSE reconnection safety net via polling

**Gaps / Missed Areas:**
- API design left implicit — no explicit endpoint shapes, idempotency keys, or response schemas
- Durability NFR conflated with implementation ("use S3" vs "11 nines")
- No load balancer, no CDN for downloads
- Metadata DB scaling at 20B rows not discussed (sharding/read replicas)
- Folder.files[] denormalization left as written
- SSE per-node connection capacity not quantified

---

## Feedback Given

| Rubric | Score | Notes |
|---|---|---|
| Requirements clarification | 3.5/5 | Proactive clarifying questions appeared. NFR numbers mostly solid; durability and latency required prompting. |
| High-level architecture | 4/5 | Strong upload flow (presigned + multipart + S3 event + reconciliation). |
| API design | 2/5 | Mostly skipped — no explicit endpoints, idempotency, or response shapes. |
| Component design & trade-offs | 4/5 | Clean service separation; Redis pub/sub for fanout; defended hybrid sync. |
| Scalability & fault tolerance | 3.5/5 | Pivoted polling→SSE on load math; missing LB, CDN, DB scaling strategy. |
| Deep dive quality | 4/5 | Multipart, reconciliation, fanout, reconnection, deletion all probed. |
| Communication | 3.5/5 | Better than coding rounds; tradeoffs explained more. |
| Diagram quality | 4/5 | Diagram tracked verbal design at each section. |

**Overall: 3.6/5**

**Recurring patterns:**
- API design consistently underspecified across sessions.
- Conflating implementation choices with NFR targets (durability).
- Denormalized array columns appearing without justification (Folder.files[], same pattern as earlier schema rounds).

**Time Taken: 56 minutes**
