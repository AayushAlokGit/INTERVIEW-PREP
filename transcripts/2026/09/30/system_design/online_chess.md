# System Design Round Transcript
**Date:** 2026-09-30
**Start Time:** 18:11:48 · **End Time:** 19:21:38 · **Duration:** 70 min
**Problem:** Online Chess Platform (chess.com / lichess)
**Difficulty:** 4/5 (Hard). Several dependent hard problems: server-authoritative clocks with network-lag compensation and timeout enforcement; stateful game servers whose in-memory state must survive a crash without the opponent seeing a move the recovered game doesn't have (ack ordering, ownership/fencing, reconnect by move number); and a 1M-spectator fan-out on a single game.
**Dominant pattern:** realtime updates (bidirectional game WebSockets + spectator fan-out), with a contention/ordering element on the per-game move sequence
**Performance Rating:** 2/5  <!-- machine-read on future rounds; ≤2 = eligible for re-ask, ≥3 retired -->

**Would it have fit a real 45-min round?** No. He would have been cut off while writing the HLD (API finished at +33, HLD submitted at +52). The deep dive would never have started.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 18 min (18:30:07) | +10 | Over by 10 min |
| Core entities | 12 min | 26 min (18:37:58) | +14 | Over by 14 min |
| API design | 17 min | 33 min (18:44:42) | +16 | Over by 16 min |
| High-level design | 27 min | 55 min (19:06:36) | +28 | Over by 28 min |
| Deep dive | 40 min | 69 min (19:20:43) | +29 | Over by 29 min |
| Wrap-up | 45 min | 70 min (19:21:38) | +25 | Over; no content |
| **Total** | 45 min | 70 min | +25 | Over |

## Round-trip Tax
| Phase | Parts asked | Answered 1st pass | Follow-up exchanges | Minutes lost | What was missing |
|---|---|---|---|---|---|
| Requirements | FRs, NFRs with numbers | FRs yes; NFRs partly | 2 (+6 single-question clarifications before any FR) | ~4 (follow-ups) + ~6 (drip-fed clarifications) | Rating update and game end (checkmate/resign/draw/timeout) not FRs. Games/s double-counted (500 vs 250). No moves/s, no peak, no connection count, no storage/yr, no consistency stance for *live* games. "Games/s" unit on a concurrency figure. |
| Core entities | 1 | Partly | 2 | ~4 | Clocks, per-time-control rating, and move ordering came only after probes. Self-added GameMatchingRequest (positive). |
| API design | 1 | Partly | 1 | ~2 | Replay endpoint and nextCursor only after probe. No top-live-games endpoint (FR 5). sendMove has no moveNumber/sequence. No resign/draw/game-over/clock server messages. No illegal-move rejection. No cancel on matching. |
| High-level design | 1 | Partly | 1 | ~3 (+19 min composing) | Inbound move path, fanout source, Postgres writer, and notification channel all missing on first pass |
| Deep dive | 6 probes | 1 partial (per-move persistence) | 5 | ~14 | Lag charged one-way only (6 s, not 12 s). Fix trusted the client timestamp. Timeout enforcement unanswered. Ack-before-persist crash unanswered. Clocks not in move event. 1M spectators unsized ("unsure"). Wrap-up "dont know". |
| **Total** | | | 11 + 6 clarifications | ~33 of 70 | |

**Deferrals used:** 0 by him. (The interviewer held the match-notification channel over to the HLD; he returned to it with a Notification System and a polling alternative.)

---

## Conversation Log
**Interviewer (18:11:48, +0m):** Presented online chess (chess.com/lichess): find an opponent, play a live timed game move by move, review past games. Difficulty 4/5 (Hard). Gave the reference timeline, measured and not enforced, and the canvas path. Asked for requirements.
**Aayush:** Full move history of past games, or just final status?
**Interviewer (+2m):** Complete move history; replay move by move.
**Aayush:** How will the matching of users be done?
**Interviewer (+2m):** Product rule: pick a time control; pair with same time control and similar rating. Mechanism is his to design.
**Aayush:** How will ratings change?
**Interviewer (+3m):** One rating per time control; updated once at game end from the result and the opponent's rating; Elo-style formula, a black box.
**Aayush:** FRs: 1. create accounts; 2. find opponent to play live timed game; 3. see past games (full history); 4. pick match type, paired by same type + similar rating.
**Interviewer (+5m):** Rendered. NFRs next, with numbers.
**Aayush:** What should the scale be?
**Interviewer (+7m):** 10M DAU · 5 games/user/day · 80 moves & 10 min per game · 100 B per move · kept forever · peak 3× · top games up to 1M live spectators.
**Aayush:** Is streaming a game also a requirement?
**Interviewer (+8m):** Live spectating yes; video no.
**Aayush:** Update the FRs to include that.
**Interviewer (+9m):** Added FR 5 (spectate live game, see moves as they happen).
**Aayush:** How do spectators choose a game: search, or list all?
**Interviewer (+10m):** No search. Direct link, or a "Top live games" list (50 most-watched).
**Aayush:** NFRs: 99.99% availability for live matches, lower OK for past games; eventual consistency OK for past games; move propagation p99 < 200 ms; 10M DAU × 5 → 500 games/s, 4 MB/s ingress; up to 1M spectators, read-heavy for move updates.
**Interviewer (+15m):** Rendered. (1) Show the 500 games/s and 4 MB/s arithmetic with day = 10^5 s. (2) Games/s is start rate; what sizes the live tier, and its peak value?
**Aayush:** 10^7 × 5 / 10^5 = 500 games/s; 80 moves = 8 KB/game → 4 MB/s. Concurrent games is the important number.
**Interviewer (+17m):** Agreed. What is it at peak?
**Aayush:** 500 × 10 × 60 = 3×10^5 "games/s" is maximum concurrent games.
**Interviewer (+18m):** Noted. Core entities.
**Aayush:** User(rating, name, id); Game(id, player1, player2, createdAt, status, winner); GameMove(gameId, playerId, chessPieceType, startPosition, endPosition).
**Interviewer (+21m) [structural]:** Where do time control, remaining clocks and per-time-control rating live? How is move order fixed for replay?
**Aayush:** Added matchType to Game, moveNumber to GameMove.
**Interviewer (+23m) [structural, 2nd]:** Remaining clocks? Per-time-control rating vs single User.rating?
**Aayush:** Clocks live in Game. New entity MatchTypeUserRating(matchType, userId, rating).
**Interviewer (+24m):** Rendered. API next.
**Aayush:** Another entity GameMatchingRequest(userId, status: MATCHED, matchedUser) is needed.
**Interviewer (+26m):** Added. API next.
**Aayush:** API: identity from auth headers. POST /gameMatchingRequests with Idempotency-Key, body {matchType}, returns GameMatchingRequest(id, userId, matchType, status: MATCHING); user notified on match. WS /games/:id: client→server sendMove(piece, start, end); server→client sendOpponentMove(piece, start, end). GET /games?cursor&limit → Game[]. GET /games/:id → SSE for move updates.
**Interviewer (+31m):** Rendered. (1) [structural] Which endpoint returns a finished game's moves? (2) [structural] How does a matched client get the gameId, and over what channel? (3) [not blocking] How does the client get the next cursor?
**Aayush:** (1) GET /games/:id/moves. (2) A notification to both players with the game link. (3) nextCursor in the response.
**Interviewer (+33m):** Updated. Notification channel held to the HLD. HLD next.
**Aayush (+52m):** Client → LB → API + WebSocket layer. Matchmaking Service uses Redis queues by match type and rating, pairs players and assigns them to a Game Server. Game Server holds board, turn and clocks in process memory and processes moves sequentially; sends accepted moves to both players through WebSockets; publishes events to Kafka for rating updates, notifications, persistence. PostgreSQL stores users, completed games, move history "so games can be recovered if a server fails". A separate spectator fanout layer distributes moves via WebSocket gateways. Alternative: spectators poll the moves DB, or SSE via Redis pub/sub, but pub/sub is at-most-once so polling is still needed.
**Interviewer (+52m):** Drew only the described arrows. (1) Inbound move path to the owning Game Server? (2) Fanout layer's source? (3) Who writes Postgres, when; does recovery hold with only completed games? (4) Match notification channel?
**Aayush:** (1) Consistent hashing routes all players of a game to one server. (2) Fanout reads moves from Kafka. (3) Game service writes moves to Postgres periodically. (4) Matching service depends on a notification system (in-app or other); alternatively a UI listing the user's matching requests with a join link once matched.
**Interviewer (+55m):** Rendered. Which part would you deep-dive first, and why?
**Aayush:** Time handling: player clocks differ, so keep the clock centrally at the game server and send remaining time in WebSocket messages.
**Interviewer (+56m):** Good pick. (1) A 300 ms RTT player in 1+0 bullet who moves instantly: time lost per move and over 40 moves vs a 20 ms player, and the fix? (2) Clock hits zero while the player is disconnected: who ends the game, and how does it know?
**Aayush:** not able to answer this
**Interviewer (+59m):** Just part 1, from your own design: when does the server start and stop this player's clock, and what happens in between?
**Aayush:** Network trips also eat into the player's time.
**Interviewer (+59m):** How much, for 40 moves of a 60 s game, and the fix?
**Aayush:** 150 ms to reach the server × 40 = 6 s of 60 s. Fix: use the timestamp on the player's machine when the move was made.
**Interviewer (+61m):** (1) Is the leg where the opponent's move travels to him also charged? (2) A modified client reports 0.01 s per move. What stops it, given client clocks can't be trusted?
**Aayush:** unsure how to answer the questions
**Interviewer (+62m):** Left open. A Game Server with 50k live games crashes: what do players see, who takes over, and where are board, moves and both clocks rebuilt from, given Postgres is written periodically?
**Aayush:** Moves written to Postgres as they arrive, by a Kafka consumer. On failure the game restarts on another server, rebuilt from the DB; spectators redirected to the new server.
**Interviewer (+64m):** Updated canvas. Sequence: server pushes White's move to Black, then crashes before it reaches Kafka. What board does the replacement rebuild, what does Black see, what next? Does the move event carry enough to rebuild both clocks?
**Aayush:** unsure how to answer the questions
**Interviewer (+65m):** Left open. 1M spectators on one game via Kafka → fanout → WS gateways: connections, gateways, what happens per move, what breaks first and the fix?
**Aayush:** A separate streaming service; spectators join the same instance depending on the games they subscribed to; instances subscribe to a Redis pub/sub channel; one push per move by the Kafka consumer, instances relay to their users.
**Interviewer (+67m):** Rendered. If one game's spectators join the same instance, how many connections does it hold? If spread out, how many instances and how many messages per move?
**Aayush:** unsrue
**Interviewer (+69m):** Name the two biggest bottlenecks or risks and what you'd trade to fix each.
**Aayush:** dont know

---

## Design Summary
**Requirements Gathered:** FRs: accounts; matchmaking by time control and rating; live timed play; replay of past games with full history; live spectating. NFRs: 99.99% for live games; eventual consistency for past games; move p99 < 200 ms; 500 games/s (should be 250); 4 MB/s ingress (should be 2 MB/s avg, 6 MB/s peak); 3×10^5 concurrent games (should be 1.5×10^5 avg, 4.5×10^5 peak); up to 1M spectators per game.
**High-Level Architecture:** Client → LB → API/WS layer → (consistent hashing) Game Server holding in-memory game state. Matchmaking Service with Redis queues assigns games to Game Servers and calls a Notification System. Game Server → Kafka → (consumer) PostgreSQL per move, and → spectator fanout / Redis pub/sub per game → streaming instances → spectators.
**Key Design Decisions & Trade-offs:** Server-authoritative clock (right). Stateful in-memory game server (right, but no ownership/failover design). Redis pub/sub vs polling for spectators (the only trade-off argued; he noted at-most-once himself). Idempotency-Key on matchmaking.
**Scalability & Fault Tolerance Points:** Per-move persistence via Kafka (added in the deep dive). Recovery by rebuilding from Postgres (lags Kafka; ack ordering unresolved).
**Gaps / Missed Areas:** Lag compensation and timeout enforcement; ack-after-durable ordering; clocks missing from the move event; game ownership/fencing and client reconnect-by-moveNumber; move idempotency (sendMove has no sequence); consistent-hash rebalancing moves live games; spectator fan-out sizing; matchmaking atomicity (double-matching), widening rating window, cancel; top-live-games list; rating-update idempotency; monitoring of any kind.

---

## Feedback Given

### Pace report
| Phase | Ref | Actual | |
|---|---|---|---|
| Requirements | 8 | 18 | over by 10 |
| Core entities | 12 | 26 | over by 14 |
| API | 17 | 33 | over by 16 |
| HLD | 27 | 55 | over by 28 |
| Deep dive | 40 | 69 | over by 29 |
| Wrap-up | 45 | 70 | over, no content |

**Would it have fit a real 45-minute round? No.** At 45 min he was still composing the HLD, which arrived at +52. A real interviewer would have cut him off in the HLD, and the deep dive (clocks, crash recovery, spectator fan-out) would never have started. The biggest sink was the **19 minutes composing the HLD** (+33 → +52). Second was the requirements phase: six single questions, each answered on its own, before any FR was written. This problem is a 4, but the overrun was **process, not problem size**. The front half alone finished 16 min over before anything hard began. Once the hard parts arrived, the gap was knowledge ("unsure" four times), not time.

**Round-trip tax:** 11 follow-up exchanges plus 6 drip-fed clarifications. That's about 33 of 70 minutes, nearly half the round, spent extracting pieces rather than designing. **Deferrals used: 0.**

### Senior-signal scorecard
| Signal | Status | Reason |
|---|---|---|
| Owns the narrative | Mixed | Self-raised: Idempotency-Key on matching, the GameMatchingRequest entity, Redis pub/sub being at-most-once, and clocks as the hard part. But the HLD shipped without an inbound move path, a Postgres writer, or a notification channel, and the deep dive went to "unsure" four times. |
| Leads with trade-offs | Weak | The only trade-off argued was pub/sub vs polling. No named alternative for in-memory state, consistent hashing, Kafka, Postgres, or Redis queues. |
| Pushes scale until it breaks | Weak | Games/s double-counted players (500 vs 250). No moves/s (20k avg / 60k peak), no peak concurrency, no connection count, no storage per year. The 1M-spectator break went unsized: "unsure". |
| API as a designed contract | Mixed | Idempotency-Key, an async id with a status, and a WS command vocabulary are good. But sendMove has no moveNumber, so a retry after reconnect can double-apply. No clock, game-over, resign or draw messages. No top-live-games endpoint. Replay and nextCursor only came after probes. |
| Operability | Weak | None raised: no crash detection, split-brain/fencing, reconnect, consumer lag, or any metric. |
| Pace | Weak | HLD finished at 55 vs 27. Round ran 70 min. Deep dive never produced a resolved break. |

**Overall:** mid-level execution on the front half, below mid-level in the deep dive. **No-hire** at senior.

**Performance Rating: 2/5.** Major gaps: NFR numbers wrong and incomplete, every scale break left unaddressed, and the deep dive unresolved on all three problems. This is above a 1 because the HLD's shape (stateful game server, matchmaking on Redis, Kafka → persistence, a separate spectator tier) was his own.

### What a senior strong-hire would have done on THIS problem
1. **Numbers that decide it, in 2 minutes:** 10M × 5 / 2 players = 25M games/day ≈ 250 games/s. × 80 moves = 2B moves/day ≈ **20k moves/s, 60k peak**. 250 × 600 s = **150k concurrent games (450k peak) → ~900k player WebSockets at peak**. 200 GB/day ≈ 70 TB/yr of moves. The sentence that decides the architecture: "Writes are tiny and modest; the hard part is ~1M long-lived stateful connections and per-game ordering, so the design is WebSockets pinned to a per-game owner, not stateless REST."
2. **Clock (the problem he picked):** the server clock runs from *send* of the opponent's move to *receipt* of this one. That charges the **full RTT**: 300 ms × 40 = 12 s of a 60 s game (20%), vs 0.8 s at 20 ms. Fix: **lag compensation**. The server measures each connection's RTT with ping/pong and credits `min(measured one-way lag, cap)` per move, with a per-game quota. It never trusts a raw client timestamp, because a cheater reports 0.01 s. **Timeout:** the owner keeps a deadline timer per game (`now + remaining` for the side to move) and flags the loss when it fires, whether or not the client is connected. There's a separate disconnect/abandon timer.
3. **Crash: persist before you ack.** Append the move (with `moveNumber`, both clocks, and the server timestamp) to the per-game log, i.e. Kafka keyed by gameId with acks=all. Only then push it to the opponent. Otherwise Black saw a move the recovered game doesn't have. The recovering owner rebuilds from the **log**, not from Postgres, which lags the consumer. Ownership goes through a lease or registry (gameId → server) with a **fencing token**, so a half-dead old owner can't write. Clients reconnect with `lastMoveNumber` and the server replays the gap. Clocks pause for the failover window.
4. **Move idempotency:** `sendMove(gameId, moveNumber, uci)`. The server applies it only if `moveNumber == expected`, returns the existing result for a duplicate, and rejects stale moves. This is the at-least-once trap on the core write.
5. **Routing trade-off:** pure consistent hashing *moves live games* when a node joins, because ~1/N of keys remap. A senior names that and picks a **registry/lease** (assign at match time, sticky until the game ends) and scales out by placing *new* games only.
6. **1M spectators:** 1M connections at ~50k per gateway ≈ 20+ gateway nodes. Each gateway subscribes **once** to the game's channel and fans out locally, giving a two-level tree: 1 publish → ~20 subscribers → 1M sockets. Add a deliberate spectator delay (anti-cheat, as lichess does) and coalesce moves. Each message carries `moveNumber`, and a gap triggers `GET /moves?after=`. That makes at-most-once pub/sub safe without polling everyone.
7. **Operability:** move-ack p99, owner heartbeat and failover time, Kafka consumer lag into Postgres, gateway connection count per node, the hot-game spectator count, and matchmaking wait time per rating band.

Checklist: see `system_design_senior_guidance.md` → "Quick pre-round self-check", especially the lines on avg **and** peak numbers, idempotency/failure traps, and pushing scale.

---

## Drill Prescription
| # | Gap targeted (signal / weakness) | Exercise (on which system) | Timebox | Pass criterion |
|---|---|---|---|---|
| 1 | Pushes scale / arithmetic slips: 500 vs 250 games/s (double-counted players), no moves/s, no peak, no connection count | Redo only the NFRs and back-of-envelope for **online chess** from the same givens. Write every arithmetic step with day = 10^5 s, and end on the one sentence that decides the architecture | 8 min | 250 games/s, 20k/60k moves/s, 150k/450k concurrent games, ~0.9M peak player sockets, ~70 TB/yr, all within 2×. Deciding sentence names long-lived stateful connections. Done in ≤ 8 min. |
| 2 | Deep dive: "unsure" on crash ordering and move retry (resilience, idempotency) | **Scale break / failure walk** on his chess design: a game owner crashes after pushing a move but before it's durable. Write the fixed sequence (persist → ack → push), the move-event fields, the ownership/fencing mechanism, and client reconnect-by-moveNumber | 15 min | Persist-before-ack stated; the move event carries moveNumber + both clocks + server ts; fencing token or lease named; duplicate/stale sendMove handling defined. No prompting. |
| 3 | Leads with trade-offs, and the clock deep dive (lag compensation) | **Trade-off table** for the server clock: (a) raw server receipt time, (b) client-reported timestamp, (c) server-measured-lag compensation with cap. For each: what it gives up, how it's cheated, and what breaks for a 300 ms player at 1+0 | 12 min | Full-RTT charge computed (12 s vs 0.8 s); client-timestamp rejected with the cheating reason; capped compensation chosen; timeout owner timer defined |

---

## Drill Follow-up (2026-09-30)
**Start:** 19:34:21 · **End:** 19:47:57 · **Duration:** 14 min (ended by Aayush during drill 3)

| # | Drill | Time (box) | Result | Notes |
|---|---|---|---|---|
| 1 | Chess NFR numbers | 6 min (8) | **Fail** | Games/s fixed (250, no double count) and concurrency right (150k games, 900k peak sockets). Moves/s 25 vs 20k (per-game rate never multiplied by concurrent games), which cascaded into ingress 2.5 KB/s vs 2 MB/s and storage 2.5 GB/day vs 200 GB/day; no per-year figure; peaks written as "3×" not numbers. Read-heavy call right but not refined to "hot games only". |
| 2 | Crash walkthrough | 4.5 min (15) | **Partial (2/4)** | Persist-to-Kafka-before-push stated unprompted (the key idea). Move event lacked moveNumber/serverTs. Takeover: "lock on the Kafka queue" — no lease holder, no failure detection, no fencing token/epoch. Reconnect: in-memory dedup on moveNumber, but no RESUME{lastMoveNumber}, no duplicate-vs-stale handling. |
| 3 | Clock trade-off table | — (12) | **Fail (not attempted)** | "not sure how to answer"; scaffolded to row (a) only (one move, 150 ms each way); drill ended before an answer. |

**Compared with the round:** real improvement on the two numbers he'd been shown (games/s, concurrency) and on persist-before-ack. The un-shown step — multiplying a per-entity rate by the population — failed, and the clock problem is still untouched after two exposures. Next attempt: redo drill 3 from row (a) cell by cell before any new round.
