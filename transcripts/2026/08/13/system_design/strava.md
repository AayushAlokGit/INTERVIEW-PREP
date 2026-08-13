# System Design Round Transcript
**Date:** 2026-08-13
**Start Time:** 10:48 · **End Time:** 11:32 · **Duration:** 43 min
**Problem:** Strava — activity tracking & segment leaderboards
**Difficulty:** Medium
**Dominant pattern:** scaling writes (GPS ingest) + proximity/geospatial (segment matching), with scaling reads on the feed
**Performance Rating:** 2/5  <!-- machine-read on future rounds; <=3 = eligible for re-ask, >=4 retired -->

**Would it have fit a real 45-min round?** No — cut off at API design. HLD and deep dive were never reached; the round ended by candidate request at 43 min with no architecture designed.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 17.5 min | +9.5 | No |
| Core entities | 12 min | 27.5 min | +15.5 | No |
| API design | 17 min | 42.0 min | +25.0 | No |
| High-level design | 27 min | not reached | — | No |
| Deep dive | 40 min | not reached | — | No |
| Wrap-up | 45 min | not reached | — | No |
| **Total** | 45 min | **43 min (front half only)** | — | No |

### Granular front-half breakdown
| Segment | Wall time | Verdict |
|---|---|---|
| FRs + out-of-scope | 4.5 min | Fine |
| NFRs + back-of-envelope | 13.0 min | Biggest single sink |
| Core entities (first draft) | 4.5 min | Fine |
| Entity probe round-trip | 5.5 min | Ended in "let's move on"; questions unanswered |
| API (first draft) | 7.0 min | Slightly slow |
| API probe round-trips (2 rounds) | 7.5 min | Second biggest sink |

~15 of the 43 minutes went to re-asking for content already inside the scope of the original question.

---

## Conversation Log

**Interviewer:** Problem: Design Strava. Users record outdoor activities (runs, rides) from a mobile app. During an activity the phone streams GPS points. When the activity ends, the user sees their route on a map with distance, pace, and elevation. Activities appear in their followers' feeds. Certain stretches of road are marked as segments, and every activity that passes over a segment gets ranked on that segment's all-time leaderboard. Difficulty: Medium. Reference timeline stated (measured, not enforced). Canvas path given. Start with requirements — FRs, then NFRs with numbers.

**Aayush:**
FRs:
0. A user can follow other users of app.
1. User records activity (runs, rides) from mobile app.
2. User location updated during activity by phone GPS coordinate stream.
3. On activity end show activity stats like distance, pace and elevation.
4. Activity details appear in feeds of a user's followers.
5. Classify certain stretches of roads as segments.
6. Activity passing through segments get ranked in an all-time leaderboard.

Out of scope:
1. Global activity leaderboard for users.

**Interviewer:** FR list is clear and the out-of-scope call is good. Two things: (1) FR 2 — "location updated during activity" — updated for whom? Purely the recording user's own device, or does someone else see it live? That changes the architecture significantly. (2) NFRs — I need numbers: total users and peak concurrent recorders; GPS emit frequency and implied write QPS; read/write ratio; p99 for feed load and leaderboard load, and leaderboard freshness after activity end; consistency vs availability and for which subsystem. Derive out loud — I want the arithmetic.

**Aayush:**
1. Watching other users' live activity is out of scope for now, so location recording is for personal tracking.

NFRs:
1. Eventual consistency acceptable for activity feed and leaderboards.
2. Highly available for activity creation and tracking (99.99% ~52 mins downtime/year).
3. Phone pings GPS coordinates every 10s.
4. Low latency feed load (p99 < 200ms), segment leaderboard up to 3 min late.
5. Assuming scale of 100M DAU, each starting 2 activities/day -> 2k activities/s. With 1M concurrent users, each pinging GPS updates every 10s -> 100k GPS updates/s. Assuming each activity is of 100Kb -> 200Mb/s for activity data. Read:write ratio of 100:1 for activities.

**Interviewer:** Good — numbers on the table. Now sanity-check them against each other, because two of them are fighting. (1) 100M DAU x 2 activities/day = 200M activities/day, and separately 1M concurrent recorders. Take a typical activity duration and reconcile those. Does 1M concurrent fall out of 200M/day, or is one wrong? Whichever way it resolves, what's the corrected GPS write rate? (2) Storage: 200M activities/day at 100KB — what's that per day and per year, and does it change where activity data lives? (3) Units check: 2k/s x 100KB — 200 Mb/s or 200 MB/s?

**Aayush:** go with 1M concurrent recorders and ignore storage for now will comeback to that later

**Interviewer:** Fine — 1M concurrent recorders, 100k GPS points/s, and we'll return to storage. Requirements locked; FRs and NFRs rendered to the canvas. Next: core entities. Two things I specifically want decided, not glossed: how is the GPS point stream modeled relative to an activity — part of the Activity object or separate, and why? And what exactly is a Segment as data — what fields does it need for the system to tell that an activity passed over it?

**Aayush:**
Core Entities:
1. User (id, name, email)
2. Activity (id, userId, distance, pace, elevation, segmentsCovered: [SegmentId], route: [GPS coordinates])
3. Segment (id, route: [GPS Coordinates])
4. FollowRelationship (fromUserId, toUserId)
5. SegmentToActivity(segmentId, activityId)

**Interviewer:** Three probes. (1) The leaderboard ranks efforts on a segment. Looking at SegmentToActivity(segmentId, activityId) — if I ask you to render segment X's leaderboard, can you? What is it sorted by, and is that value present anywhere in your model? (2) Activity.route embeds the point stream — is that column in the same store as distance/pace, and what's the alternative you're rejecting? (3) Given only Segment.route, how would the system find which segments an activity's route crosses? Just tell me whether the field is sufficient, and if not, what's missing.

**Aayush:**
1. Need to also add the column which will be used to sort the activity in the leaderboard to the SegmentToActivity model.
2. We can store the GPS coordinates in a WAL log DB like Cassandra and store reference to the object in the DB where distance, pace are being stored.
3. Field seems to be insufficient but it would need something to cheaply compare 2 lists of time-ordered GPS coordinates.

**Interviewer:** (1) Name it. What's the actual field on SegmentToActivity, its type, and how is it computed from the raw GPS stream? (2) Good — Cassandra for the stream, reference in metadata store. What's the partition key and clustering key on that table, and what did you consider instead of Cassandra (object storage? time-series DB?) and why reject it? (3) Right that it's insufficient — what extra field(s) on Segment make the cheap compare possible at all?

**Aayush:** lets move on

**Interviewer:** Moving on. Entities rendered as described. API design — verbs, paths, request bodies, explicit response shapes with named fields. Cover at minimum: starting a recording and streaming GPS points; ending an activity; fetching a single activity with its route; the follower feed; the segment leaderboard. Decide as you go which need pagination and which need idempotency, and say why.

**Aayush:**
NOTE: Identity extracted from auth headers
1. PUT /follow/:userId — Response: 2xx status
2. POST /activity — Request: {startTime, startLocation}; Response: Activity(id, userId, startLocation, status); Idempotency key header
3. PUT /activity/:id/locationUpdate — Request: {gpsCoordinate}; Response: 2xx status
4. PUT /activity/:id/end — Response: 2xx status to end activity
5. GET /activities?cursor={}&limit={} — Response: Activity(id, distance, userId, pace)[], nextCursor. Cursor based pagination for dynamic feeds
6. GET /segments/:id/leaderboard?cursor={}&limit={} — Response: Activity(id, distance, userId, pace)[], nextCursor. Cursor based pagination for dynamic feeds

**Interviewer:** Solid contract — cursor pagination justified, auth from headers, idempotency key present. Four things to fix or defend. (1) locationUpdate with a singular gpsCoordinate is one HTTP request per point per user = 100k req/s at your numbers. Defend or change it. And your idempotency key is on POST /activity, which fires once per activity; this endpoint fires 360 times over a flaky mobile network. What are the retry semantics, and what happens if the client retries a point or delivers two out of order? (2) Missing: fetching a single activity with its route. What is it, and what's in the response given the route lives in a separate store? (3) GET /activities — my own activities or my follower feed? Disambiguate and give me the feed one. (4) The leaderboard returns {id, distance, userId, pace}. Is that enough to render the screen? Name what a client needs that you haven't returned.

**Aayush:** how would the leaderboard ranking be decided?

**Interviewer:** That's your call, not mine — I'd rather see you reason to it. The segment is a fixed stretch of road, and an activity crosses it with timestamped GPS points. What's the natural thing to rank athletes by on a fixed stretch of road, and how do you derive it from the points? Then answer 1–4.

**Aayush:**
1. There is need of a batch endpoint. There is need of idempotency key header in the PUT endpoint as well to handle multiple client retries or client queuing up and sending batch requests for GPS updates due to temporary network issues. In order to avoid write bloat on client retries we need to have deduplicating mechanisms in this endpoint.
2. GET /activity/:id -> Activity(id, distance, pace, route:[], ..)
3. GET /activities/:followedUserId -> refinement
4. The leaderboard response will have a duration as well, and the Activity entity will also have this duration along with the SegmentToActivity.

**Interviewer:** Duration on the segment is the right answer — that's the effort time and it's what the leaderboard sorts on. Two quick closes. (1) GET /activities/:followedUserId is "show me one person's activities." FR 4 is one merged, time-ordered stream across everyone I follow. Give me that endpoint's path and response shape. (2) Dedup on the batch GPS write — what's the actual dedup key that makes a retried batch a no-op?

**Aayush:**
1. GET /friends/activities
2. timestamp

**Interviewer:** API rendered to the canvas. High-level design: components, datastores, queues, caches, directional flows, anchored on the API. Take the paths one at a time — GPS ingest from 1M concurrent phones; activity end to stats displayed; feed served inside 200ms p99; activity to leaderboard entry and the leaderboard read. Name concrete systems, and for each significant box the named alternative you rejected.

**Aayush:** how much time have i taken?

**Interviewer:** 43 minutes elapsed. Requirements through API took the whole reference budget; HLD and deep dive are both still ahead. The clock isn't a gate — we'll run the design to its end and I'll report pace honestly at the debrief. Go ahead with the HLD.

**Aayush:** no wait lets end this here i want to analyse my weakpoints in this part of the design i have always noticed myself taking too much time here

**Interviewer:** [Round ended at candidate's request. Full debrief given — see Feedback Given.]

---

## Design Summary

**Requirements Gathered:**
- 7 FRs covering follow graph, activity recording, GPS streaming, end-of-activity stats, follower feed, segment definition, segment leaderboard.
- Out of scope (stated): global cross-user leaderboard; live spectating of another user's activity.
- NFRs: eventual consistency for feed + leaderboards; 99.99% availability for activity create/track; GPS ping every 10s; feed p99 < 200ms; leaderboard staleness budget 3 min; read:write 100:1.
- BoE: 100M DAU x 2/day = 200M activities/day ≈ 2k/s; 1M concurrent x 1 ping/10s = 100k GPS points/s; ~100KB/activity ≈ 200 MB/s.
- Unresolved: 1M concurrent contradicts 200M/day at realistic activity duration (~5M concurrent implied, ~5x the ingest estimate). Storage/day and /year deferred and never revisited.

**High-Level Architecture:** Not produced. Round ended before the HLD phase.

**Key Design Decisions & Trade-offs:**
- GPS point stream stored separately from activity metadata: Cassandra for the stream, reference held in the metadata store. No alternative named (object storage / TSDB not discussed); partition and clustering keys not specified.
- Cursor-based pagination on feed and leaderboard, justified by the dataset being dynamic/changing.
- Batch GPS ingest endpoint with an Idempotency-Key header; dedup key = timestamp (implicitly per activity).
- `duration` added as the leaderboard sort field, carried on both Activity and SegmentToActivity.

**Scalability & Fault Tolerance Points:** None reached. No ingest tier, queueing, fan-out strategy, caching, replication, or failure handling was designed.

**Gaps / Missed Areas:**
- Entire HLD and deep dive missing.
- Segment matching algorithm — the core hard problem — never touched; candidate identified the field gap on Segment but declined to propose a fix.
- Scale inconsistency (1M vs ~5M concurrent) raised by interviewer and abandoned.
- Storage sizing (20 TB/day, ~7 PB/yr) never computed despite an explicit promise to return to it.
- Units slip (Mb vs MB) unaddressed.
- No named alternative for any component in the entire round.
- Feed endpoint given as a path with no response shape.
- Cassandra partition/clustering key unspecified.
- Out-of-order GPS point delivery raised by interviewer, never answered.
- Operability entirely unexamined: hot segments, consumer lag, tie-breaks, GPS noise/cheating, monitoring, cost.

---

## Feedback Given

**Time Taken: 43 minutes** — all of it in the front half.

### 0. Pace report

| Phase | Reference | Actual (cumulative) | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 17.5 min | +9.5 | No |
| Core entities | 12 min | 27.5 min | +15.5 | No |
| API design | 17 min | 42.0 min | +25.0 | No |
| High-level design | 27 min | never reached | — | No |
| Deep dive | 40 min | never reached | — | No |
| Wrap-up | 45 min | — | — | No |
| **Total** | 45 min | 43 min (front half only) | — | No |

**Would this have fit a real 45-minute round? No.** The entire round went to requirements, entities, and API. A real interviewer's round would have ended at minute 45 with zero architecture on the table — no ingest path, no feed, no leaderboard, no deep dive. That is a no-hire regardless of how good the front half was, because the front half is not what's being graded. Requirements/entities/API are the setup; the interview is the HLD and the deep dive.

**Where the 43 minutes went:**

| Segment | Wall time | Verdict |
|---|---|---|
| FRs (7 lines + out-of-scope) | 4.5 min | Fine — this part is efficient |
| NFRs + back-of-envelope | 13.0 min | Single biggest sink |
| Core entities (first draft) | 4.5 min | Fine |
| Entity probe round-trip | 5.5 min | Ended in "let's move on" — 5.5 min spent, questions still unanswered |
| API (first draft) | 7.0 min | Acceptable, slightly slow |
| API probe round-trips (2 rounds) | 7.5 min | Second biggest sink |

Two distinct failure modes:

**Sink 1 — NFRs took 13 minutes to produce 5 lines.** The content was good: real numbers, a stated read/write ratio, a differentiated consistency stance, a latency budget split between feed and leaderboard. That's a 3-minute answer. 13 minutes means it was being derived rather than walked. The NFR checklist is finite and known before you see the problem — availability, consistency, latency, scale, read/write ratio, durability. Walk it top-to-bottom at speed, state a number for each, explicitly skip the ones that don't bind. You don't need to think about which NFRs exist; only about the numbers.

**Sink 2 — ~70% of an answer emitted, the remaining 30% costing a full round-trip.** The pattern across the round: no sort column on SegmentToActivity -> asked -> "need to add the column" -> asked again for the name -> "let's move on." Single-point endpoint at 100k/s -> asked -> correct batching + idempotency + dedup answer. Leaderboard response missing the ranking field -> asked -> candidate asked the interviewer what it should be -> pushed back -> `duration` produced immediately. Feed endpoint -> per-user path -> asked -> merged path but still no response shape, the one thing originally requested. Six round-trips, 2–4 min each: roughly 15 of the 43 minutes were spent re-asking for things already in scope of the original question. That, not slow thinking, is what eats the front half.

The tell was the leaderboard exchange: the ranking question was asked of the interviewer, and when refused, answered correctly in one line. Reaching for the interviewer instead of committing to an attempt cost a round-trip on a known answer.

### 1. Senior-signal scorecard

| Signal | Read | Why |
|---|---|---|
| Owns the narrative (self-raises traps) | Weak | Every trap came from the interviewer: the 100k req/s endpoint, the missing leaderboard sort key, the merged-feed shape, the dedup key. Asked the interviewer for the ranking answer outright. |
| Leads with trade-offs vs alternatives | Weak | One named component all round (Cassandra); asked what was rejected in its favour, the question went unanswered. No alternative named anywhere. |
| Pushes scale until it breaks | Weak | The opposite: two mutually inconsistent numbers (1M concurrent vs 200M activities/day — at ~40 min each that's ~5M concurrent, 5x the ingest estimate), declined to reconcile, deferred storage, never returned. 200M x 100KB = 20 TB/day, ~7 PB/year — a number that decides where activity data lives, never said. |
| API as a designed contract | Mixed (round's high point) | Genuinely good instincts: auth from headers, idempotency key on create, cursor pagination with a stated reason, correct verbs. Marked down for gaps needing extraction — singular point endpoint, missing GET /activity/:id, no response shape on the feed. |
| Operability / second-order concerns | Not observed | Never reached. No hot partitions, lag, monitoring, or cost. |
| Pace | Weak | 43 minutes, no architecture. |

**Overall: mid-level, no-hire on this round.** Not because the thinking is bad — the API section shows senior instincts and every corrected answer was correct. Because 43 minutes bought a data model and a URL list, and an interviewer scores what got designed.

### 2. Performance Rating: 2/5

Weak. NFR numbers present and a decent API contract, but no architecture at all and the scale inconsistency was raised and abandoned. Eligible for re-ask.

### 3. What a senior strong-hire would have done on this problem

On the clock: FRs + NFRs by minute 6, entities by 10, API by 16, and the remaining 29 minutes on the two things Strava actually turns on. Front-half phases are throughput exercises — be fast there precisely because they're rehearsable.

Traps to self-raise, unprompted:
- "One PUT per GPS point is 100k req/s — absurd for the API tier, so the client buffers and posts a batch every 30–60s, which also handles tunnels and dead zones. Batches retry, so each point carries (activityId, timestamp) and the write is an idempotent upsert." All of this came out — after prompting.
- "1M concurrent doesn't square with 200M activities/day; at ~40 min each that's ~5M concurrent and my ingest is 5x what I said. I'll design for 500k points/s." Catching your own inconsistent traffic model is among the cheapest senior signals available; it was handed over and passed on.
- "20 TB/day of raw GPS is not going in Cassandra rows — the polyline goes to S3 as a compressed blob keyed by activityId, metadata in Postgres, hot recent window in Cassandra."

Alternatives never named: Cassandra vs object storage vs TSDB for the point stream. Fan-out-on-write vs fan-out-on-read for the feed — and the hybrid, because a Strava pro with 500k followers is the celebrity problem in a fitness app. Redis ZSET vs indexed SQL ORDER BY for the leaderboard.

The hard problem never reached — segment matching. At ~2k finished activities/s against a global segment table, "which segments did this route cross?" is not a list-compare. A senior gets to: index segment start points in a geospatial index (S2/geohash cells, Redis GEO or PostGIS), take candidate segments whose cell the route touches, verify the route actually traverses the segment polyline in order and within tolerance, emit an effort with its duration. Then the scale-break: this runs async off a Kafka topic on activity-end, not inline, because at 2k/s inline matching blows the latency budget — which raises consumer lag, DLQs for un-matchable routes, and "leaderboard 3 min stale" as a derived SLA rather than an asserted one. That conversation is the round, and it never started.

Operability, all unexamined: a viral segment (Alpe d'Huez) is a hot key on both the ZSET and the matching worker; tie-breaking on equal durations; GPS noise and cheating (a "run" at 60 km/h); how you'd know a leaderboard had gone stale or wrong.

Drill: run `/design-sprint` on a few systems — front half only, timeboxed, which is exactly the muscle that failed here. Target: FRs+NFRs in 6 minutes, entities in 4, API in 6, complete on the first pass with zero follow-ups needed.

Checklist to review before the next round: `system_design_senior_guidance.md`.
