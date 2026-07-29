# System Design Round Transcript
**Date:** 2026-06-22
**Start Time:** 14:23
**End Time:** 15:14
**Duration:** 51 minutes
**Problem:** Design a Live Streaming Platform (Twitch-like)

---

## Conversation Log

**Interviewer:** Presented the prompt — live streaming (Twitch/YouTube Live): streamers broadcast live, viewers watch low-latency, viewer counts, live chat. Set a pacing goal: skeleton (reqs → entities → API → HLD) by ~22 min, candidate drives the deep dives. Asked him to drive requirements and numbers.

**Aayush:** Start time 14:23. FRs: (1) streamer broadcasts real-time video, (2) viewers watch live, (3) streamers see viewer counts, (4) live chat. NFRs: HA 99.9% (~8.6 hrs/yr), low latency (initially "p99 < 200ms, video reaches all viewers within 200ms"), eventual consistency OK for viewer count and chat, scale 100M DAU with 1M concurrent viewers per stream.

**Interviewer:** Flagged two things: (1) 200ms glass-to-glass is WebRTC-grade and forks the whole design vs HLS — commit deliberately; (2) compute the headline number — egress bandwidth for a 1M-viewer stream at ~5 Mbps, plus ingest.

**Aayush:** Revised latency to "latest frame visible within ~1s." Computed egress = 1M × 5 Mbps = **5 Tbps** for a single stream.

**Interviewer:** Confirmed 5 Tbps correct; noted the 1s + 5 Tbps tension is the deep dive. Rendered reqs to canvas. Asked for entities + API in one pass, with explicit response shapes and real-time-push vs REST.

**Aayush:** Entities: Video (live bytes), LiveStreamMetadata (id, name, streamingUserId, startedAt), User, StreamViewer (id, userId, joinedAt, videoId), ChatMessage (id, streamViewerId, videoId, txt). API: POST /liveStreams (Idempotency-Key) → presigned URL to upload video to blob storage; WS /livestream/:id for live chat (sendMessage / pushMessages).

**Interviewer:** Flagged the ingest as a VOD pattern (presigned URL → blob storage can't meet 1s live latency); asked how video actually enters in real time. Also flagged the missing viewer playback path.

**Aayush:** Corrected: can't afford streamer→S3→client-poll round trip within 1s; video bytes should pass through our servers which fan out to viewers; no recording so drop bytes if no subscribers. Viewers use a subscribe/watch endpoint per stream id.

**Interviewer:** Rendered entities + API (marked presigned URL REVISED). Asked for the full HLD skeleton with delivery protocol + fan-out mechanism named, choices justified.

**Aayush:** Streamer sends bytes over WebSocket (or UDP for speed, accepting some loss). Single server forwards bytes to subscribers; drop if none. Subscribe = WS connection + DB record of viewer. Viewer count = separate API, polled, does a DB COUNT query, scaled with indices. Chat over the same WS, server fans out; messages may be lost if server dies. Self-flagged: single server is a bottleneck / breaks at high viewer counts.

**Interviewer:** Rendered naive HLD. Noted he self-raised the breakages. Pushed the fan-out deep dive: one server can't hold 1M WS connections or push 5 Tbps — design the broken case (topology + protocol); and does every viewer get identical bytes (ABR)?

**Aayush:** Split reception from fan-out. Reception service pushes bytes to Kafka partitioned by streamId (ordering/durability). Fan-out service consumes partitions, multiple instances behind LB; viewers assigned an instance via consistent hashing on streamId (all viewers of a stream → same instance); instance holds viewer connections and fans out. If too many viewers for one instance, distribute across servers. Also transcode into multiple qualities; fan-out service uses client-reported network capacity to pick quality (ABR).

**Interviewer:** Surfaced the contradiction: consistent hashing on streamId pins all 1M viewers to ONE instance — a hot partition, the same bottleneck. Asked how to spread one stream's viewers across many servers, and how 5 Tbps reaches worldwide viewers in 1s (missing component).

**Aayush:** Partition by streamId + geography to split better.

**Interviewer:** Geography → regional edge servers (good, missing component). But within a region 200K viewers still exceed one server, and there's only one ingest copy — how do bytes replicate from single ingest to thousands of edges without any single node fanning out to all? Describe the tiers.

**Aayush:** A tree structure — leaf nodes are edge servers, root is the ingest service, multiple relay servers in between; each node's egress load is reduced (e.g. a branching tree spreads/halves per-node fan-out).

**Interviewer:** Confirmed — fan-out tree / CDN tiered distribution; total 5 Tbps distributed across edges near viewers for the 1s budget. Second deep dive: viewer count — DB COUNT + per-viewer rows won't scale (1M join/leave churn, hot aggregation). Given eventual-consistency/approximate is OK and edge servers know their connection counts, what's cheaper?

**Aayush:** Aggregate each edge server's connection count periodically; no need to store viewer records in the DB at all.

**Interviewer:** Confirmed — approximate, eventually-consistent count via edge aggregation, drops the write churn. Noted strong pace (~37 min, two deep dives). Moved to feedback.

**Aayush:** End time 15:14.

---

## Design Summary
**Requirements Gathered:** FRs — broadcast live, watch live, viewer counts, live chat. NFRs — HA 99.9%; latest frame within ~1s; eventual consistency for count + chat; 100M DAU, 1M concurrent/stream; egress ~5 Tbps for one 1M-viewer stream (computed correctly). Latency target refined 200ms → 1s.

**Core Entities:** Video, LiveStreamMetadata, User, StreamViewer (later dropped), ChatMessage.

**High-Level Architecture:** v1 (naive, self-flagged): streamer → single Stream Server (WS/UDP) → fan out bytes to viewer WS connections; drop if no subscribers; DB viewer records; polled COUNT for viewer count; chat over same WS. Evolved in deep dive to: ingest service → Kafka (partition by streamId) → fan-out service tier → **fan-out tree with regional edge servers** → viewers; ABR transcoding; edge-aggregated approximate viewer count.

**Key Design Decisions & Trade-offs:** WebSocket for chat (real-time push); idempotency key on create; UDP vs packet-loss trade-off named; Kafka for ordered/durable ingest; consistent hashing (led to hot-key contradiction); fan-out tree for replication.

**Scalability & Fault Tolerance Points:** Split ingest from fan-out; fan-out tree to spread 5 Tbps; regional edges for geography/latency; edge-aggregated viewer count removing DB churn; acknowledged byte loss on server failure (no recording).

**Gaps / Missed Areas:** Vague API response shapes; missing/loose viewer playback endpoint; VOD-style ingest misconception (corrected after prompt); did not self-catch the consistent-hashing hot-key contradiction; protocol fork (LL-HLS vs WebRTC) not made explicit at requirements time; transcode lodged inside fan-out rather than a separate tier; chat-at-scale not scaled (reused video WS, no pub-sub backplane / rate limiting); operability (CDN egress cost, ingest redundancy, regional failover, backpressure) light.

---

## Feedback Given

**Requirements (Strong):** Fast, decisive, numbers present; 5 Tbps egress computed correctly (no arithmetic slip); over-aggressive 200ms corrected to defensible 1s when pushed. Best requirements phase in a while.

**Entities (Good):** Quick, reasonable; StreamViewer DB entity later self-dropped in favor of edge aggregation.

**API (Mixed — weakest phase):** Idempotency key + WebSocket-for-chat good self-raised calls. But presigned-URL→blob ingest was a VOD mental model (fixed after prompt); viewer playback endpoint omitted until prompted; watch endpoint left vague (no verb/path/response shape); thin response shapes (recurring).

**HLD (Good narrative):** v1 naive but candidate flagged its own breakages unprompted — set up his own deep dive.

**Deep dives (Mixed→Good):** Drove to fan-out tree + ABR + edge-aggregated count (correct destinations), but introduced and did not self-catch the consistent-hashing hot-key contradiction, and needed escalating nudges to reach the tree.

### Senior-Signal Scorecard
| Signal | Status | Reason |
|---|---|---|
| Owns the narrative / self-raises traps | Mixed (improving) | Self-raised WS-chat, bottleneck, ABR; missed self-catching hot-key contradiction + ingest misconception |
| Leads with trade-offs vs named alternatives | Mixed | Good UDP-vs-loss trade-off; Kafka / consistent-hashing / tree stated without alternatives |
| Pushes scale until it breaks | Mixed (improving) | Engaged the break and designed the fixed case, but was led to the contradiction and the tree |
| API as a designed contract | Mixed | Idempotency + WS good; vague response shapes, missing playback endpoint, VOD-ingest error |
| Operability / second-order | Mixed (improving) | Edge-aggregated count a genuinely good insight; missed CDN cost, backpressure, regional failover |
| Pace (core by mid, deep dive after) | Strong | Skeleton by ~22 min, two deep dives, finished ~51 min — big jump from last round |

**Level read:** Solid hire-leaning round — strongest recently, mainly because pace flipped Weak→Strong and he self-raised his own breakages. To reach strong-hire: catch his own contradictions (hot key was sitting there) and justify each box against a named alternative in the same breath.

### What a senior strong-hire would have done on THIS problem
- Named the protocol fork at requirements time: 1s to 1M viewers forces LL-HLS over CDN (~2s realistic) vs WebRTC (sub-second, expensive SFU trees, no CDN cache) — pick and own the trade-off.
- Self-surfaced the hot key: hashing viewers by streamId pins them to one node → need a fan-out tree, not a single owner.
- Separated transcode (ABR ladder 1080/720/480/360) from fan-out cleanly: ingest → transcode → packager → CDN tree.
- Owned chat-at-scale as its own hot key: shard chat servers + Redis/Kafka pub-sub backplane + rate limiting/slow-mode.
- Operability: CDN egress as the dominant cost (5 Tbps), ingest redundancy, regional failover.

**Time Taken: 51 minutes**
