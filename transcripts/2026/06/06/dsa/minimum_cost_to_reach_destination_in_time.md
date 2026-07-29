# DSA Round Transcript
**Date:** 2026-06-06
**Start Time:** 9:27
**End Time:** 10:17
**Duration:** 50 minutes
**Problem:** Minimum Cost to Reach Destination in Time
**Topic:** Graphs / DP over (node, resource) state (constrained shortest path)
**Difficulty:** Hard

---

## Problem Statement
A country of `n` cities (0..n-1) connected by bidirectional roads. `edges[i] = [x_i, y_i, time_i]` (multiple roads of differing times allowed between a pair; no self-loops). `passingFees[j]` is paid every time you pass through city `j`, including start city 0 and destination city n-1. Start at city 0 with `maxTime` minutes; total travel time must not exceed `maxTime`. Return minimum total passing fees from 0 to n-1 within budget, or -1 if impossible.

**Example:** maxTime=30, edges=[[0,1,10],[1,2,10],[2,5,10],[0,3,1],[3,4,10],[4,5,15]], passingFees=[5,1,2,20,20,3], n=6 → 11 (path 0→1→2→5: time 30, fees 5+1+2+3=11; the time-cheaper 0→3→4→5 costs fees 48).

Constraints: 1≤n≤1000, n-1≤edges≤1000, 1≤time_i,maxTime≤1000, 1≤passingFees[j]≤1000; no duplicate (pair,time); bidirectional; may be disconnected.

---

## Conversation Log

**Interviewer:** Presented problem; asked for clarifications + approach before coding.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints (n≤1000, edges≤1000, times/fees 1..1000, multi-edge, bidirectional, possibly disconnected).

**Aayush:** Brute force — DFS from src tracking currentTime, enumerate all paths to dest within maxTime, take min passing fare across valid paths.

**Interviewer:** (1) Complexity of brute force? (2) Where's the waste — when are two arrivals at the same city equivalent? Define the state to memoize.

**Aayush:** (1) Exponential, each node has edges to up to n-1 others.

**Interviewer:** Right. Now Q2 — define the state.

**Aayush:** State is (node, timeUsed), value = minFee for this state.

**Interviewer:** Correct — and note you can't collapse to just node (slower-cheaper vs faster-pricier both useful). Lock: (1) #states, (2) the transition/recurrence, (3) answer extraction + -1 case. Top-down or bottom-up?

**Aayush:** States node≤1000 × time≤1000. solve(node, current_time) = min passing fee to reach dest from node given current_time. Base: if current_time>maxTime return INF; if node==dest return 0. Iterate neighbors; ans = min(ans, solve(nbr, cur+w) + passingFees[node]).

**Interviewer:** Hand-evaluate your recurrence on path 0→1→2→5. What total — is it 11? Which fee got dropped?

**Aayush:** When node==dest, add passingFees[node].

**Interviewer:** Correct (5+1+2+3=11). Now complexity + confirm memo.

**Aayush:** TC O(n*maxTime), Space O(n*maxTime) DP + O(n) recursion stack.

**Interviewer:** Time drops the edge factor — per state you iterate neighbors. Σ degree × maxTime = O(maxTime·E). Stack depth is O(maxTime) not O(n) (cycles allowed). Now code it.

**Aayush:** [Submitted memoized DFS with a `parent` param + `if(nbr==parent) continue;`, dp[node][time], prints raw INT_MAX on unreachable.]

**Interviewer:** Prints 11 — good. Two issues. (1) `parent`: is it necessary (what guarantees termination)? And it's not in the memo key, so a result cached for one parent is reused for another → breaks memo when (node,time) reachable via two parents. (2) Unreachable returns INT_MAX, must be -1.

**Aayush:** Remove parent and simply return -1.

**Interviewer:** Right — but apply the -1 only at the top-level result in main; keep solve returning INT_MAX internally (the `nbrAns < INT_MAX` logic depends on it; returning -1 from solve would be added as a real fee). With those fixes it's correct and optimal.

**Aayush:** [Acknowledged.] (End time 10:17.)

---

## Solution

**Aayush's Final Solution (corrected, C++):**
```cpp
int solve(vector<vector<pair<int,int>>>& g, vector<int>& fees,
          int node, int time, int maxTime, int dest, vector<vector<int>>& dp) {
    if (time > maxTime) return INT_MAX;
    if (node == dest)   return fees[dest];
    if (dp[node][time] != -1) return dp[node][time];
    int ans = INT_MAX;
    for (auto [nbr, wt] : g[node]) {                 // parent skip removed
        int sub = solve(g, fees, nbr, time + wt, maxTime, dest, dp);
        if (sub < INT_MAX) ans = min(ans, sub + fees[node]);
    }
    return dp[node][time] = ans;
}
// main:
int ans = solve(graph, passingFees, 0, 0, maxTime, dest, dp);
cout << (ans == INT_MAX ? -1 : ans);                 // -1 only at top level
```
(Bugs fixed during round: dest-fee base case [caught pre-code via trace]; `parent` param breaking memoization; missing INT_MAX→-1.)

**Time Complexity:** O(maxTime · E) — corrected from an initial O(n·maxTime).
**Space Complexity:** O(n · maxTime) memo; O(maxTime) recursion stack.

---

## Feedback Given

**Problem Understanding & Clarification — Good.** Asked for constraints up front (consistent two rounds); registered bidirectional/multi-edge/disconnected.

**Approach & Thought Process — Strong (best part).** Brute-force first, then self-drove the `(node, timeUsed)` state and explicitly reasoned why it can't collapse to just `node` — exactly the lesson the prior round had to drag out of him. Clear transfer of learning.

**Code Quality & Correctness — Mixed, improved.** Caught the destination-fee base-case bug BEFORE coding by hand-tracing solve(0,0) (the target habit). Two bugs remained: (1) `parent` param — unnecessary tree-DFS reflex that silently breaks memoization (key omits parent); (2) no INT_MAX→-1 (repeat of last round). Single example test masked the parent bug (optimal path never revisits).

**Complexity Analysis — Mixed.** First O(n·maxTime) dropped the edge factor; corrected to O(maxTime·E). Called stack O(n) instead of O(maxTime). Space correct.

**Communication — Good (improved).** Engaged with traces instead of asserting past them; pace 50 min (down from 79).

**Time Taken: 50 minutes** (down from 79 last round; bar ~40–45).

**One-line takeaway:** Strong step up — self-drove the state insight and traced the fee bug pre-code. Stubborn recurring misses to bank as a pre-submit checklist: unreachable→-1, and "test the input that breaks it, not the one that passes."
