# System Design Round Transcript
**Date:** 2026-08-22
**Start Time:** 14:28:22 · **End Time:** 14:57:21 · **Duration:** 29 min (abandoned at requirements)
**Problem:** LLM Serving Platform (ChatGPT-style inference service)
**Difficulty:** 5/5 (Hard) — several independent hard problems that cannot all fit 45 minutes: stateful GPU workers holding KV cache for the life of a stream, continuous batching, tokens/sec (not QPS) as the unit of capacity, multi-tier admission control under hard scarcity, cold-start of weights measured in minutes, and long-lived streaming connections with reconnect/cancel semantics
**Dominant pattern:** long-running tasks + realtime updates (token streaming), with a scaling-writes/capacity break on stateful GPU scheduling
**Performance Rating:** 1/5  <!-- machine-read on future rounds; ≤2 = eligible for re-ask, ≥3 retired -->

**Would it have fit a real 45-min round?** No — the round stopped inside the requirements phase. At minute 15 there were four FRs, no out-of-scope list, and no NFRs. Nothing past requirements was reached.

## Phase Timings
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Requirements | 8 min | 15 min (FRs only; NFRs never produced) | +7 and incomplete | No |
| Core entities | 12 min | not reached | — | — |
| API design | 17 min | not reached | — | — |
| High-level design | 27 min | not reached | — | — |
| Deep dive | 40 min | not reached | — | — |
| Wrap-up | 45 min | not reached | — | — |
| **Total** | 45 min | 29 min elapsed, abandoned | — | No |

## Round-trip Tax
| Phase | Parts asked | Answered 1st pass | Follow-up exchanges | Minutes lost | What was missing |
|---|---|---|---|---|---|
| Requirements | 3 | 1 | 1 (unanswered) | ~14 | Out-of-scope list; all NFRs; every number (TTFT, TPOT, total generation time, scale, peak ratio, token-based capacity model) |
| Core entities | — | — | — | — | Phase never reached |
| API design | — | — | — | — | Phase never reached |
| High-level design | — | — | — | — | Phase never reached |
| Deep dive | — | — | — | — | Phase never reached |
| **Total** | 3 | 1 | 1 | ~14 | — |

**Deferrals used:** 0 — none offered, none taken.

---

## Conversation Log

**Interviewer [14:28:22]:** Design an LLM Serving Platform. You're building the inference serving layer behind a ChatGPT-style product. A user sends a conversation (a chat history plus a new message) to your service. The service must run it through a large language model and stream the generated tokens back to the user's browser as they are produced. Users expect the first token to appear fast and the rest to flow smoothly. The models run on GPUs, which are expensive, scarce, and slow to provision. Multiple model sizes are offered, and paying tiers get better service than free ones. Design the system end to end — from the user's request to the tokens on their screen. **Difficulty: 5/5 (Hard)**. [45-min reference timeline stated, measured not enforced; canvas path given.] Start with requirements — FRs, what's explicitly out of scope, then NFRs with concrete numbers. Ask me for any scale inputs you need.

**Aayush [14:33:49]:** What does better service mean? Will we have to handle the loading of model weights onto the GPU as well as provisioning the GPU?

**Interviewer [14:33:49]:** Better service means: when GPU capacity is saturated and requests are queuing, paid users get served ahead of free users, and free users can be rate-limited or shed entirely. Paid tier has a tighter time-to-first-token target; free tier is best-effort. How you express that is yours to design. Model weights and GPU provisioning: yes, both in scope — assume you own the fleet. A GPU node comes up with no model loaded; loading weights takes ~1 minute for the small model and several minutes for the large one. Provisioning a brand-new GPU node from the cloud provider takes several minutes on top of that, and capacity is not always available. You decide how much of that you design versus scope out — just say which.

**Aayush [14:43:17]:** FRs ->
1. A user sends a conversation (a chat history plus a new message) to your service. The service must run it through a large language model and stream the generated tokens back to the user's browser as they are produced.
2. Output tokens should be streamed to user as they are generated.
3. A ChatCompletion request is handled by models which run on GPUs which are expensive, scarce and slow to provision. System must provision GPUs if needed and load model weights into GPUs for inference.
4. Multiple model sizes offered, multiple tiers of service - paying tier better service, compared to free tier.

**Interviewer [14:43:17]:** Noted. Two things before NFRs: (1) What are you explicitly putting out of scope? Name them. (2) NFRs with concrete numbers — latency targets (be specific about which latency; this system has more than one that matters), scale, and read/write shape. Ask me for any givens you want.

**Aayush [14:57:21]:** debrief

---

## Design Summary

**Requirements Gathered:**
- Four FRs, three of which restate the prompt verbatim. FR1 and FR2 are the same requirement written twice (streaming). FR3 and FR4 are constraints supplied by the interviewer rather than decisions made by the candidate.
- Two clarifying questions asked before writing anything — both well-chosen (definition of tiering; ownership of provisioning + weight-loading). This was the strongest moment of the round.
- No out-of-scope list.
- No NFRs of any kind: no TTFT, no inter-token latency, no total generation time, no scale figures, no peak multiplier, no capacity unit, no availability or durability position.
- Never asked for scale givens despite being invited to twice.

**High-Level Architecture:** None produced.

**Key Design Decisions & Trade-offs:** None produced.

**Scalability & Fault Tolerance Points:** None produced.

**Gaps / Missed Areas:**
- Cancellation of an in-flight generation — a first-class FR on this problem (users hit stop constantly; it is live GPU spend) — never raised.
- Reconnect/resume of a dropped stream mid-generation — never raised.
- The capacity-unit trap (QPS is the wrong unit; GPU capacity is tokens/sec and requests differ ~100x in cost) — never reached.
- The three distinct latency SLOs (TTFT, TPOT, total generation) and their mutual trade-off — never reached.
- Statefulness of inference workers (KV cache resident in GPU memory for the life of a stream) and everything that follows from it — never reached.
- Admission control as memory admission; tier-based queueing and shedding — never reached.

---

## Feedback Given

### 0. Pace report
At minute 15 there were four FRs and no NFRs. A real interviewer would have been at the 8-minute marker with nothing numeric to anchor a design against, with entities, API, HLD and deep dive all still ahead. The round did not run out of time; it stopped. Biggest time sink: 10 minutes producing four FRs, three of which are the problem statement read back. On a 5/5 this is fatal — and the overrun is process, not problem size. Difficulty never bound, because the round never got far enough for it to.

Round-trip tax: 3 parts asked in requirements, 1 answered, 1 follow-up left unanswered. Zero deferrals.

### 1. Senior-signal scorecard
- **Owns the narrative — Weak.** Two clarifying questions, then FRs echoing the prompt. No trap self-raised, though this problem hands over several for free (KV-cache statefulness, tokens/sec as the unit of load, what happens to a half-streamed response when a GPU dies).
- **Leads with trade-offs vs alternatives — Not observed.** No design produced.
- **Pushes scale until it breaks — Weak.** Nothing sized. On this problem the scale break *is* the interview.
- **API as a designed contract — Not observed.**
- **Operability / second-order — Not observed.**
- **Pace — Weak.** 15 min for an incomplete requirements phase, then abandonment.

**Overall read:** No-hire on this sitting — not because the design was wrong, but because there was no design. Mid-level at best on available evidence. Credit where due: the two clarifying questions were exactly the right two scoping questions on this problem, asked before writing anything. That instinct is senior. Then the momentum stopped.

### 2. Performance Rating: 1/5 — Poor
No coherent design was produced. Eligible for re-ask.

### 3. What a senior strong-hire would have done on THIS problem

**On the FRs.** Shorter, and making actual choices rather than restating the prompt:
- Submit a conversation, get a streamed completion.
- Resume/reconnect a stream that dropped mid-generation.
- Cancel an in-flight generation (real money on a GPU).
- Route to a requested model; enforce per-tier admission and priority.
- Out of scope, stated aloud: training/fine-tuning, conversation persistence, the model itself, safety filtering, billing/metering.

Cancellation and reconnect are the two FRs separating people who have thought about streaming inference from people who haven't. Neither appeared.

**On the NFRs.** This system has three distinct latency numbers and a senior states all three, because they trade against each other:
- TTFT (time to first token) — the perceived responsiveness number. p99 < 1s paid, < 5s free.
- TPOT / inter-token latency — must beat reading speed, ~50ms/token, or the stream stutters.
- Total generation time — a 500-token answer at 50ms/token is 25s of *held GPU*, which is what makes this unlike a normal request/response system.

Then the load model, and the trap never reached: **QPS is the wrong unit.** GPU capacity is tokens/second, and requests differ by ~100x in cost. Sizing this fleet in requests/sec produces a fleet wrong by an order of magnitude. Deriving that unprompted is the highest-value move available in the first 10 minutes.

Givens the interviewer was holding: 100M DAU · 10 messages/user/day · 1,000 input and 500 output tokens per message · peak 3x average · a large-model GPU generates ~2,000 output tokens/sec aggregate across concurrent requests · 90% of traffic to the small model. (Day = 10^5 seconds.) That arithmetic yields thousands of GPUs at a cost dwarfing everything else — which makes batching, tiering, and KV-cache reuse obviously necessary rather than decorative.

**Where the design breaks — where the round was going.** Continuous batching: a GPU serves many concurrent streams by batching each token step. So a "server" is not stateless — each in-flight conversation holds a KV cache of GBs in GPU memory for the whole generation. Consequences: you cannot load-balance round-robin; a node loss destroys every in-flight stream on it; admission control is really *memory* admission (will this KV cache fit); long-context requests are the hot key. That is the senior conversation on this problem, reachable in 45 minutes only if requirements take 8.

**Operability that would have been asked for:** queue depth as the alert (not CPU), per-tier TTFT degradation monitoring, what gets shed first when capacity is gone, and cost per million tokens at the derived fleet size.

### 4. Checklist
Review `system_design_senior_guidance.md` — specifically §1 (own the narrative), §5/NFR checklist (every NFR needs a number and a context), and the note that a round is won by finishing the core with time left, not by polishing the front.
