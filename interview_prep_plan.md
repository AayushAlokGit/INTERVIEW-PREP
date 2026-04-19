# Interview Preparation Plan

## Where You Stand

**Strengths:** Distributed systems, backend (C#, Django), Azure cloud, AI/LLM exposure, strong impact stories with metrics.

**Gaps for US market:** DSA consistency, system design breadth, frontend depth (TypeScript, modern React patterns), SQL at interview depth.

---

## The 4 Pillars

### 1. DSA / Coding (Highest Priority)

US companies — from startups to FAANG — filter almost universally on this. At your level (3.5 yrs), you're expected to solve medium problems cleanly and hard problems partially.

**Target:** 100–120 problems over 6–8 weeks.

**Topic order (by ROI):**
1. Arrays, Strings, Hashmaps — week 1
2. Two Pointers, Sliding Window — week 1–2
3. Binary Search — week 2
4. Trees (BFS/DFS) — week 2–3
5. Linked Lists, Stack/Queue — week 3
6. Graphs (BFS/DFS, Union-Find) — week 4
7. Dynamic Programming (1D → 2D) — week 5–6
8. Heaps, Intervals — week 6

**Resource:** NeetCode 150 — curated for exactly your experience level.

**Daily habit:** 2 problems/day on weekdays, 1 timed mock on weekends.

---

### 2. System Design (High Priority at 3+ years)

You have real material here — telemetry pipelines, billing infrastructure. The goal is to verbalize it in interview format.

**Core concepts to cover:**
- API design (REST, pagination, rate limiting, idempotency)
- Database choice (SQL vs NoSQL tradeoffs, indexing, sharding)
- Caching (Redis, cache invalidation, TTL)
- Message queues / async processing (Kafka, pub/sub)
- Horizontal scaling, load balancing
- CAP theorem, consistency models
- Monitoring and observability

**Classic problems to practice:**
- URL shortener
- Rate limiter
- Notification system
- Feed/timeline
- File upload service
- Distributed job scheduler

**Resource:** "System Design Interview" by Alex Xu (Vol 1 + 2).

---

### 3. Behavioral / Leadership (Medium — but differentiating)

You have strong stories. The work here is structuring them tightly in STAR format and having 5–6 ready for any question type.

**Stories to build around:**
- Payroll automation ($45K recovered) → ownership, impact
- Billing infrastructure across timezones → complexity, cross-team
- Churn validations framework (20% ticket reduction) → collaboration, customer empathy
- COGS migration ($156K/year) → cost thinking, technical judgment
- MCP server side project → initiative, GenAI depth

**Question types to prepare for:**
- Most impactful project
- Disagreement with a teammate
- Ambiguous requirements, shipped anyway
- Failure / what you'd do differently
- Why are you leaving your current role

---

### 4. Language & Stack Fundamentals (Lower priority but don't skip)

US startups often do live coding in a specific stack. You should be comfortable across:

- **Python** — most interview-friendly language, good fallback
- **TypeScript/JavaScript** — async/await, event loop, closures, types
- **SQL** — joins, subqueries, window functions, GROUP BY logic
- **React** — hooks, state management, component lifecycle, useEffect pitfalls

---

## 6-Week Execution Plan

| Week | Primary | Secondary |
|------|---------|-----------|
| 1 | DSA: Arrays, Strings, Hashmaps | SQL: joins, GROUP BY, window functions |
| 2 | DSA: Two Pointers, Sliding Window, Binary Search | Behavioral: draft 5 STAR stories |
| 3 | DSA: Trees, Linked Lists, Stack | System Design: API design + caching |
| 4 | DSA: Graphs, Union-Find | System Design: DB design + queues |
| 5 | DSA: DP (1D first, then 2D) | Mock interviews — DSA + system design |
| 6 | DSA: Heaps, Intervals, review weak areas | Full mock loops + behavioral polish |

---

## Mock Interview Tools

| Command | Use it for |
|---------|-----------|
| `/dsa-round` | Timed LeetCode-style coding round |
| `/system-design` | Full system design mock with probing |
| `/behavioral` | STAR round with follow-ups |
| `/mock-interview` | Full loop (all three combined) |
| `/interview-feedback` | Structured debrief after any session |

---

## Daily Routine (Full-Time Prep, ~9 hrs/day)

Structure every day into 4 blocks. Don't skip breaks — retention drops sharply after 90 mins of focused work.

### Weekday Schedule

| Time | Block | Activity |
|------|-------|----------|
| 8:00 – 8:30 | Warmup | Review yesterday's DSA problems — read solutions, trace through edge cases |
| 8:30 – 10:30 | DSA Block 1 | Solve 2 new problems (timed: 30–45 min each). No peeking until time is up. |
| 10:30 – 10:45 | Break | Walk, stretch |
| 10:45 – 12:15 | DSA Block 2 | Deep-dive 1 problem: read editorial, rewrite cleanly, note the pattern |
| 12:15 – 1:15 | Lunch | Full break — no screens |
| 1:15 – 2:45 | System Design | Read 1 chapter (Alex Xu) or design 1 classic problem end-to-end on paper/whiteboard |
| 2:45 – 3:00 | Break | |
| 3:00 – 4:00 | Fundamentals | Rotate daily: Python → SQL → TypeScript/JS → React (one per day, cycle through) |
| 4:00 – 5:00 | Behavioral | Write or refine 1 STAR story. Say it out loud. Time yourself (under 2 min). |
| 5:00 – 6:00 | AI Coding | See rotation below — build, experiment, or study one AI/LLM concept per day |
| 6:00 – 7:00 | Mock / Review | Every other day: run a `/dsa-round` or `/behavioral` mock. Off days: review notes. |
| 7:00+ | Off | Hard stop. Sleep is when patterns consolidate. |

### Weekend Schedule

Weekends are for **integration** — putting it all together, not learning new material.

| Time | Block | Activity |
|------|-------|----------|
| 9:00 – 11:00 | Full mock round | `/mock-interview` or `/system-design` — treat it like a real interview |
| 11:00 – 12:00 | Debrief | `/interview-feedback` — identify 1–2 specific things to improve next week |
| 12:00 – 1:00 | Lunch | |
| 1:00 – 3:00 | Weak area review | Revisit the topic where you struggled most that week |
| 3:00+ | Off | Rest fully — burnout is the #1 risk in full-time prep |

---

### Daily Fundamentals Rotation

Cycle through these one per day so nothing gets stale:

| Day | Fundamentals Focus |
|-----|--------------------|
| Mon | Python — data structures, built-ins, list comprehensions, itertools |
| Tue | SQL — joins, subqueries, window functions, GROUP BY edge cases |
| Wed | TypeScript/JS — async/await, event loop, closures, generics |
| Thu | React — hooks (useState, useEffect, useMemo), component patterns |
| Fri | Free pick — whatever feels weakest that week |

---

### Weekly Targets

| Metric | Target |
|--------|--------|
| DSA problems solved | 14–16 (2–3/day weekdays) |
| System design problems | 3–4 (1 major + chapters) |
| STAR stories polished | 1–2 new or refined |
| Mock rounds completed | 2–3 full mocks |

---

### Anti-patterns to Avoid

- **Reading solutions too early** — struggle for the full time limit first, always
- **Skipping behavioral** — at 3.5 yrs experience, it's a real filter
- **Grinding the same topic** — move on even if imperfect; you'll revisit
- **No rest days** — full-time prep without rest leads to burnout within 2 weeks
