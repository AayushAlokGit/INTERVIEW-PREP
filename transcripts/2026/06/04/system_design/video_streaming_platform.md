# System Design Round Transcript
**Date:** 2026-06-04
**Start Time:** 17:18
**End Time:** 18:13
**Duration:** 55 minutes
**Problem:** Design a Video Streaming Platform (YouTube-like)

---

## Conversation Log

**Interviewer:** Welcome. Note the current time. Problem: Design a video streaming platform (YouTube-like) — users upload videos, others search/browse/watch on demand across devices and network conditions. Start by gathering requirements (functional + non-functional with numbers). Don't jump to architecture.

**Aayush:** 17:18.

**Interviewer:** Start time recorded. Go ahead — functional requirements and clarifying questions.

**Aayush:** Can users search and watch all videos?

**Interviewer:** Yes — public platform, any video public by default, any user can search/watch any video. No private/unlisted, subscriptions, or fine-grained permissions. Keep going.

**Aayush:** We'd search videos using text phrases, right?

**Interviewer:** Yes — text-based search against title/description/tags; keyword relevance, not ML ranking. But drive this yourself: state the scope and assumptions rather than checking each one. Lay out functional scope and the non-functional numbers (DAU, uploads/day, watches/day, sizes, latency, availability).

**Aayush:**
FRs: 1) Users can upload video. 2) Users can search uploaded videos by text query. 3) Users can watch uploaded videos.
Out of scope: access privileges / video sharing.
NFRs: 1) HA search 99.99% (~52 min/yr). 2) Low-latency query p99 < 200ms. 3) Durability for uploads (S3). 4) Uploads resilient to network drops. 5) Streaming resilient to drops (quality adjusted to network — transcoding). 6) Low-latency streaming p99 TTFF < 200ms. 7) 100M DAU, 10 views/day + 1 upload/day each → 10k streams/s, 1k uploads/s. High read traffic.

**Interviewer:** Good — numbers up front. But: (1) 100M uploads/day is ~30× reality; revise or own it. (2) You sized request rates but not storage or egress — compute new storage/day & /year and egress bandwidth (assume avg length + bitrate). (3) You gave average stream rate — what's peak?

**Aayush:** 1) Reduce upload assumption to realistic. 3) 10×. 2) Assuming 100MB videos and 3M uploads/day → 300TB/day in S3. Assuming standard bandwidth 100Mbps, to process 10k streams/s we'd need 1Tbps.

**Interviewer:** 300TB/day is right. But "100 Mbps per stream" — is that video bitrate or connection speed? A 1080p stream is single-digit Mbps. Re-derive egress. Also: storage multiplier from renditions? And peak stream rate / peak egress?

**Aayush:** Adjusted storage depends on quality types. Not sure what egress bandwidth really is — could you clarify?

**Interviewer:** Egress = concurrent streams × bitrate per stream. 1080p ≈ 5 Mbps. Your "100 Mbps" was a connection speed. Compute egress at avg (~10k streams) and peak (100k). For storage, assume ~5 renditions ≈ 1.5× raw and commit to a number.

**Aayush:** 50 Gbps avg and 500 Gbps peak.

**Interviewer:** Correct. That 500 Gbps is the number that decides your delivery tier. Storage with renditions ≈ 450TB/day, ~160PB/yr. Requirements locked. Next: core entities — no architecture yet.

**Aayush:** "update the drawio file"

**Interviewer:** Tell me what to draw — I draw only what you specify. We're on entities; do you want the entity list drawn? Either way, give me the core entities and fields.

**Aayush:** "add FRs and NFRs"

**Interviewer:** [Added FRs and NFRs blocks to the canvas.] Now — your core entities and key fields, especially video metadata, renditions, and what search indexes.

**Aayush:** 1) VideoMetadata 2) VideoBytes 3) User. I'll add attributes as we progress.

**Interviewer:** Give me key fields for VideoMetadata now — identity/ownership, searchable text, processing state, and how a viewer finds the playable bytes (renditions). Where does the rendition mapping live?

**Aayush:** VideoMetadata(id, userId, s3Urls:{"360p":".."...}, status: pending | uploaded, name, description).

**Interviewer:** Works — renditions as a map, status flag. (Note: uploaded ≠ watchable, transcoding sits between.) Next: API design — concrete verbs/paths, request + response shapes, idempotency/pagination/multi-step flows. Cover upload (resilient to drops), search, watch.

**Aayush:**
Identity from JWT.
POST /videos → create record (status pending). Req {videoMetadata}. Resp: presigned S3 URL.
GET /videos?query=&page=&limit= → Resp: Video(id, s3Urls, name, description)[].
GET /videos/:id → Resp: Video(id, s3Urls, name, description).
Bytes go client→S3 directly via presigned URL using service's identity, keeping app servers off the upload path.

**Interviewer:** Strong instinct. Three follow-ups: (1) single PUT of 100MB+ — re-read NFR #4, is it good enough? (2) How does status ever become watchable / what triggers transcoding? (3) Client retries POST /videos on flaky network — what happens?

**Aayush:**
POST /videos → create record (pending), status→uploaded via S3 events. Idempotency-Key header.
PUT /videos/:id → S3 multipart upload (client chunks file, resumable). Req {s3-multipart-upload-session-id}. VideoMetadata now stores multipart-upload-session-id.
(Search & watch endpoints unchanged.) Bytes client→S3 directly, servers off path.

**Interviewer:** That closes all three (resumable multipart, S3 event, idempotency key). Now the HLD end-to-end: upload+transcode, search (p99<200ms over 100M+), watch (500 Gbps peak — is S3-direct your delivery path?).

**Aayush:** "update the drawio file with core entities and APIs"

**Interviewer:** [Added Core Entities and API blocks to canvas.] Now narrate the architecture — no more diagram-deferring. Upload+transcode, search, watch.

**Aayush:** Upload: client POSTs metadata → API gateway (auth) → video service creates SQL record (pending), returns presigned S3 URL signed with service creds. Client chunks video (~5MB) and starts multipart upload to S3, gets sessionId, calls PUT to update BE. On S3 upload completion, chunks reassemble in S3 and fire a notification event; transcoding workers and DB-updating workers listen and update status + initiate transcoding pipeline. Multipart + sessionId in DB makes uploads resilient — client can query sessionId and resume. Servers off upload path; client can parallelize chunk uploads to maximize bandwidth.

**Interviewer:** Clean upload story. Two things: (a) status enum pending|uploaded — when do renditions populate s3Urls, is video playable before then? (b) What's the queue between S3 event and workers, and what happens to backlog if uploads spike? Then give me search and watch.

**Aayush:** Video becomes watchable only after transcoding completes. S3 notification events go into a Kafka queue for durability and at-least-once delivery. Partition Kafka by userId so a user's uploads land in the same partition. Transcoding workers listen to partitions, process idempotently, update DB record (state + s3Urls).

**Interviewer:** Good — Kafka durable at-least-once, idempotent workers. Challenge: why partition by userId (independent jobs, hot partition for prolific creators) vs videoId? Pick and justify. But I still need search and watch — both now.

**Aayush:** Search: client GET → video service searches metadata for watchable videos matching the query, returns records. To make text query efficient, inverted indices on name/description columns (reduces query time, increases write cost). Better: use ElasticSearch over the metadata; keep the ES index in sync with the DB via CDC events from SQL; service queries the index for optimal latency.

**Interviewer:** Right shape — ES fed by CDC, index-sync and write-amplification trade-offs named. Now the watch path at 500 Gbps peak, global audience, p99 TTFF<200ms: is S3-direct right on latency and cost? What component, and what does GET /videos/:id return for ABR to work?

**Aayush:** "updte the drawio file with the diagram described so far"

**Interviewer:** [Drew upload, transcode, and search flows exactly as described — client→gateway→service→SQL DB; direct multipart to S3; S3→Kafka→transcoding+DB-update workers→S3/DB; CDC→ElasticSearch. No watch/CDN since not yet designed.] We're at ~40 min, third diagram-deferral — give me the watch path verbally.

**Aayush:** Watch: client GETs a video, video service returns metadata; client uses s3Urls to download directly from S3. Client detects network conditions via local software and decides which resolution to download, requesting bytes for the appropriate quality so it can stream under variable network. To further reduce latency, cache video in a CDN closer to users.

**Interviewer:** You landed CDN + client ABR. Deep dive (last ~8 min): (a) With whole-file URLs, how does the client switch 1080p→480p mid-playback? What must the transcoding pipeline output, and what should GET /videos/:id return instead? (b) The scale that breaks transcoding: 3M videos/day, and a 2-hour 4K upload on one worker takes forever — how do you transcode one large video fast, and how does it interact with (a)?

**Aayush:** Segment video into chunks and transcode those chunks (video was already chunked at upload). Save mapping of quality+chunkId → s3Url for each quality. When the network drops mid-playback, the client knows which quality+chunk to retrieve and from where.

**Interviewer:** That's the key move — segment, fan out across workers, and the quality+chunkId→s3Url map is effectively your manifest (what GET /videos/:id should return). (Nuance: upload byte-chunks ≠ keyframe-aligned playback segments.) Last probe: with (N segments × 5 renditions) jobs across the fleet — how do you know the whole video is done to flip to watchable, and what if one segment's job fails / its worker dies?

**Aayush:** Tracking status of per quality + chunk.

**Interviewer:** Right idea — per-(quality,chunk) ledger, flip to watchable when completed == expected. But you left the failure half light (Kafka redelivery, DLQ for poison segment, idempotent counting). We're at time — note the time.

**Aayush:** 18:13.

**Interviewer:** [Delivered feedback + Senior Readiness debrief.]

---

## Design Summary
**Requirements Gathered:**
- FRs: upload video; text search over all videos; watch/stream video. Out of scope: access privileges/sharing.
- NFRs: HA search 99.99%; query p99<200ms; durable uploads (S3); uploads resilient to drops; adaptive-bitrate streaming resilient to drops; streaming TTFF p99<200ms.
- Scale: 100M DAU; 3M uploads/day (~1k/s); 10k streams/s avg, 100k/s peak (10×); storage ~450TB/day (~160PB/yr w/ renditions); egress 50 Gbps avg / 500 Gbps peak.

**High-Level Architecture:**
- Client → API Gateway (JWT auth) → Video Service → SQL DB (VideoMetadata source of truth).
- Upload: POST /videos creates pending record + returns presigned S3 URL; client does resumable multipart upload directly to S3 (session ID persisted); S3 upload-complete event → Kafka (partitioned by userId) → idempotent transcoding workers + DB-update workers.
- Transcode: segment video, fan out (segment × rendition) jobs across worker fleet; write renditions to S3; per-(quality,chunk) status ledger; flip to watchable when complete; store quality+chunkId→s3Url manifest.
- Search: ElasticSearch index kept in sync with SQL via CDC; Video Service queries index.
- Watch: client gets manifest, performs client-side ABR by requesting per-segment renditions; CDN caches segments near users.

**Key Design Decisions & Trade-offs:**
- Presigned URL + client→S3 direct upload to keep app servers off the data path.
- Multipart resumable upload + persisted session ID for network-drop resilience.
- Idempotency-Key on POST /videos.
- Kafka for durable at-least-once event delivery; idempotent workers.
- ElasticSearch (vs SQL inverted index) for search latency, accepting CDC-sync complexity and write amplification.
- Segmentation + parallel transcode for large videos; manifest-based ABR.

**Scalability & Fault Tolerance Points Covered:**
- Storage/egress sized; peak multiplier applied.
- Segmentation enables parallel transcode of large videos.
- Idempotent processing under at-least-once.
- CDN for delivery offload (raised late).
- Per-chunk completion tracking (failure handling left thin).

**Gaps / Missed Areas:**
- Could not size egress without help; per-stream bitrate confusion (100 Mbps vs ~5 Mbps).
- Partition-key choice (userId vs videoId) not justified; hot-partition risk unaddressed.
- CDN framed as optional optimization; egress cost argument never made.
- Transcode backpressure at peak not addressed.
- Failure handling for failed segment jobs (DLQ, retry, idempotent counting) left open.
- No monitoring/alerting or cost-at-scale discussion.
- Upload byte-chunks vs keyframe-aligned playback segments conflated.
- Repeatedly deferred answers to diagram updates; over time budget (55 min).

---

## Feedback Given

**Time Taken: 55 minutes** (target 45–50 — over budget)

### Standard Feedback
1. **Requirements — Good (improved):** FRs + NFRs with numbers stated unprompted (availability, latency, scale). Slips: unrealistic initial upload assumption, didn't size storage/egress until prompted, asked interviewer to define egress, 100 Mbps bitrate error (20× off before correction).
2. **Core Entities — Adequate:** reasonable set, renditions as s3Urls map good; wanted to defer attributes, needed prompting; missing duration/tags/timestamps/views.
3. **API Design — Strong (best part):** presigned URL, bytes off app path, multipart/resumable with persisted session, idempotency key, S3-event status, pagination. Handled all historically-missed traps — but only when pointed at each.
4. **High-Level Architecture — Solid:** complete across all four flows by the end; components named concretely.
5. **Component Design & Trade-offs — Mixed:** good ES-vs-SQL trade-off; weak partition-key justification; CDN framed as optional, no egress-cost argument.
6. **Scalability & Fault Tolerance — Mixed:** reached segmentation+parallel transcode (senior answer) but interviewer drove every escalation; fault tolerance thin (completion tracking one-liner, failure half open).
7. **Deep Dive — Good content, led delivery:** right ABR/transcode mechanics, but break→fix arc supplied by interviewer; failure fix never came.
8. **Communication — Mixed:** numbers-up-front improved; routed answers into "update the drawio" 3–4 times (deferral); needed multiple prompts; 55-min runtime.

### Senior Readiness Debrief
**Scorecard:**
- Owns the narrative / self-raises traps: **Mixed** — resolved traps well but only when prompted.
- Leads with trade-offs vs alternatives: **Mixed** — nailed ES-vs-SQL; whiffed partition key; CDN no cost trade-off.
- Pushes scale until it breaks: **Mixed** — reached the break but interviewer drove every escalation; couldn't size egress unaided.
- API as a designed contract: **Strong** — resumable multipart, idempotency, pagination, event-driven status.
- Operability / second-order concerns: **Weak** — backpressure unanswered, completion-tracking thin, no failure handling/monitoring/cost.
- Pace: **Weak** — 55 min, over budget; time lost to diagram detours.

**Overall:** Mid-to-senior boundary → **"Hire," not "Strong-Hire"** this round. Senior-grade artifacts (resumable upload, segmented ABR, parallel transcode) but the round was extracted, not driven; regression vs last two rounds on narrative-ownership, scale-pushing, and pace.

**What a senior strong-hire would have done on THIS problem:**
1. Sized egress unprompted as the headline number and framed CDN as load-bearing + cost-dominant.
2. Self-raised the four traps (single-PUT resilience, uploaded≠watchable, at-least-once idempotency, whole-file-URL ABR break).
3. Pushed own scale to the break (2-hour 4K transcode) and fully designed the distributed-completion/failure ledger (count==expected, Kafka redelivery + DLQ, idempotent counting).
4. Owned operability: transcode backpressure at peak, hot creator partition, CDN cache-hit ratio + cost, monitoring for lag/corruption.
5. Held the clock — locked decisions, avoided diagram detours, finished failure-handling.

**Drill:** re-read system_design_senior_guidance.md §1 (own the narrative), §3 (you push scale), §5 (operability). API contract design (§4) is now a strength — protect it, redirect energy upstream.
