# DSA Round Transcript
**Date:** 2026-06-06
**Start Time:** 7:59
**End Time:** 9:18
**Duration:** 79 minutes
**Problem:** Cheapest Flights Within K Stops
**Topic:** Graphs / Bellman-Ford (bounded-stops shortest path)
**Difficulty:** Medium-Hard

---

## Problem Statement
There are `n` cities connected by some number of flights. Given `flights[i] = [from_i, to_i, price_i]` (directed), and integers `src`, `dst`, `k`, return the cheapest price from `src` to `dst` with at most `k` stops, or `-1` if no such route exists.

**Example 1:** n=4, flights=[[0,1,100],[1,2,100],[2,0,100],[1,3,600],[2,3,200]], src=0, dst=3, k=1 → 700 (0→1→3; the cheaper 0→1→2→3=400 uses 2 stops > k).
**Example 2:** n=3, flights=[[0,1,100],[1,2,100],[0,2,500]], src=0, dst=2, k=0 → 500.

Constraints: 1≤n≤100, 0≤flights.length≤n(n-1)/2, 1≤price≤1e4, directed, no duplicate flights, 0≤k≤n-1.

---

## Conversation Log

**Interviewer:** Presented problem; asked for clarifying questions and approach before coding.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints (n≤100, E≤~4950, directed graph, prices 1..1e4, no duplicate edges).

**Aayush:** dst can be at most k levels from src. BFS from src up to max K levels deep. Init dist[node]=inf except src. Explore neighbors; if dist via current edge is lower, push neighbor to queue, else don't. After K levels, if dst still inf → -1.

**Interviewer:** Pin down two things. (1) Precision: k = stops; how many edges/levels? (2) The relaxation rule — single mutable dist[] within a level, danger? Need previous level frozen?

**Aayush:** k+1 edges. No issues — push only if better path found, recorded in dist[node]. When popping a node, if its dist from src > dist[node], skip it (stale), like Dijkstra skips unnecessary nodes.

**Interviewer:** [Trap.] Trace your algorithm on n=4, flights=[[0,1,100],[0,2,500],[1,2,100],[2,3,100]], src=0,dst=3,k=1. Correct answer is 600 (0→2→3, 1 stop). The 0→1→2→3=300 uses 2 stops (illegal). What does dist[2] hold and what does dist[3] become?

**Aayush:** Algorithm returns -1 since it won't relax 2→3 before k runs out.

**Interviewer:** Slow down — trace step by step. Step1 process node 0 (dist[1],dist[2]?); Step2 process node 1 (dist[2]?); Step3 process node 2 (what dist[2] read, dist[3]?).

**Aayush:** d[1]=100, d[2]=500. S2: process node 1, d[2]=200, push (2,200). Popping node 2 (val 500): 500 < ... skip it (stale). S3: level 2, processing {2,200}, not dest, skip.

**Interviewer:** Your trace gives -1, but -1 is WRONG — 0→2→3=600 is a valid 1-stop route. Your Dijkstra stale-skip threw away the fewer-stops path (500 to node 2) because the cheaper 200 (more stops) shadowed it. dist[] only remembers cost, not stops. Is "cheapest cost to reach a node" a complete state? What second piece does the state need?

**Aayush:** State should also carry number of stops so far. Stop pushing only at the K+1 level. Each queue node = (node, dist, stops). At most k+1 levels.

**Interviewer:** Right. Two things before coding: (1) Pruning — drop global stale-skip; frozen snapshot of previous level vs mutating array? (2) Complexity?

**Aayush:** No blowup since edges bounded. (2) k+1 levels, n-1 nodes each → O(kn) time and space.

**Interviewer:** Recount per-level work — a node has up to n-1 outgoing edges; total E. Restate time in terms of E. Space — do you need O(kn)?

**Aayush:** n-1 nodes × n-1 relaxations → k*(n-1)*(n-1) ~ kE time, O(n) space.

**Interviewer:** Correct — write the code.

**Aayush:** [Submitted code with `for(int i=0;i<bfsQ.size();i++)`, single dist[], k hardcoded=2, prints dist[dest].]

**Interviewer:** You hardcoded k=2 but the tricky case is k=1 (answer 600). Dry-run the inner loop `for(i=0;i<bfsQ.size();i++)` — you pop and push inside it. Does it process exactly one level?

**Aayush:** [Proposes a `level==k && u!=dest` guard instead of addressing the size issue.]

**Interviewer:** [Walked through numerically: after i=0 pops node0 and pushes 2, size=2, so i=1 continues and processes node 1 inside the level-0 pass — level boundary bleeds, yielding 300 for k=1.] One-line fix? And what does code print when dst unreachable?

**Aayush:** [Resubmitted: added `int levelSize=bfsQ.size();` snapshot AND `if(level==k && u!=dest) continue;`. Still prints dist[dest].]

**Interviewer:** levelSize snapshot is right. But the new guard is a bug — trace k=1: at level=1 both nodes 1 and 2 get skipped → dist[3]=INT_MAX. k stops = k+1 edges, so you MUST relax at level k. Also you never convert INT_MAX→-1.

**Aayush:** Remove the level==k guard.

**Interviewer:** Correct. And the unreachable case — one-line fix?

**Aayush:** Return -1 for INT_MAX.

**Interviewer:** Correct — `return dist[dest]==INT_MAX ? -1 : dist[dest];`. Algorithm now correct.

---

## Solution

**Aayush's Final Solution (corrected, C++):**
```cpp
vector<pair<int,int>> g[n];
for (auto v : flights) g[v[0]].push_back({v[1], v[2]});

queue<pair<int,int>> bfsQ;
vector<int> dist(n, INT_MAX);
dist[src] = 0;
bfsQ.push({src, 0});
int level = 0;
while (level <= k) {
    int levelSize = bfsQ.size();          // FIX 1: snapshot level boundary
    for (int i = 0; i < levelSize; i++) {
        auto vec = bfsQ.front(); bfsQ.pop();
        int u = vec.first, distance = vec.second;
        for (auto p : g[u]) {
            int v = p.first, wt = p.second;
            if (dist[v] > distance + wt) {
                dist[v] = distance + wt;
                bfsQ.push({v, dist[v]});
            }
        }
    }
    level++;
}
return dist[dest] == INT_MAX ? -1 : dist[dest];   // FIX 2: -1 on unreachable
```
(Bugs fixed during round: non-snapshotted `bfsQ.size()` loop bound; an erroneous `level==k && u!=dest` guard; missing INT_MAX→-1.)

**Optimal Solution (canonical Bellman-Ford):**
```cpp
int findCheapestPrice(int n, vector<vector<int>>& flights, int src, int dst, int k) {
    vector<int> dist(n, INT_MAX);
    dist[src] = 0;
    for (int i = 0; i <= k; i++) {        // k+1 relaxation rounds
        vector<int> tmp = dist;           // frozen snapshot
        for (auto& f : flights) {
            int u = f[0], v = f[1], w = f[2];
            if (dist[u] != INT_MAX && dist[u] + w < tmp[v])
                tmp[v] = dist[u] + w;
        }
        dist = tmp;
    }
    return dist[dst] == INT_MAX ? -1 : dist[dst];
}
```

**Time Complexity:** O(k·E) (= O(k·n²) worst case) — corrected from an initial O(kn).
**Space Complexity:** O(n).

---

## Feedback Given

**Problem Understanding & Clarification — Good (improved).** Asked for constraints up front (a historical gap), absorbed directedness. Could have asked about multigraph/negative prices but asked the key question.

**Approach & Thought Process — Mixed.** Right family immediately (level-bounded Bellman-Ford), but reached for Dijkstra's stale-skip — the signature trap. Didn't self-detect that the stop budget adds a second state dimension; needed a counterexample. Recovered correctly (state must carry stops) once shown, but the recovery was led.

**Code Quality & Correctness — Weak.** Three bugs, none self-caught: (1) `for(i=0;i<bfsQ.size();i++)` not snapshotted → level boundary bleed → stop-budget violation; (2) erroneous `level==k && u!=dest` guard (off-by-one, stops vs edges) → killed the last legal edge; (3) no INT_MAX→-1 conversion. Also hardcoded k=2, the value that masks bugs 1 & 2 instead of testing the boundary k=1.

**Complexity Analysis — Mixed.** First said O(kn), dropping the edge factor; corrected to O(k·E) with a nudge. Space O(n) correct.

**Communication — Mixed; pace poor.** Explained clearly and recovered well with evidence, but repeatedly asserted conclusions (claimed return -1; claimed the guard was the fix) instead of tracing. 79 minutes — roughly double the bar for one medium-hard.

**Time Taken: 79 minutes** (target ~40–45; over by ~35).

**One-line takeaway:** Conceptual recovery was solid; the round was decided by execution discipline — trace before asserting, and test the boundary input that breaks the code, not the one that passes.
