# DSA Round Transcript
**Date:** 2026-08-06
**Start Time:** 11:59:09 · **End Time:** 12:44:31 · **Duration:** 45 min
**Problem:** Bus Routes (LC 815)
**Topic:** Graph / BFS — 0-cost component contraction, nodes = routes not stops, adjacency by inversion
**Difficulty:** Medium-Hard
**Performance Rating:** 2/5  <!-- machine-read on future rounds; <=3 = eligible for re-ask, >=4 retired -->
**Hints Used:** 1/2
**Constraints Asked:** Open "what are the constraints?" at 12:00; semantic question "is target guaranteed to be in any route?" at 12:30 (unprompted, and acted on immediately with an early `return -1`) · **Never Asked:** whether `source` is guaranteed to be in a route; whether `source` can equal `target` (was volunteered — counted as the hint); whether a route can be a single stop (Example 2 contains `[6]`); did not use the "values unique within a route" constraint he was given
**Self-Verified:** Partial — did not trace before submitting. When given the named counterexample `source = 1, target = 1`, he located the `source == target` bug in one pass and stated a fix. The fix was incomplete and he did not notice; two further defects went uncaught.

## Phase Timings
| Phase | Budget | Actual | Hit? |
|---|---|---|---|
| Clarify | 4 min (12:03) | 2 min (12:00) | Yes |
| Approach + dry run | 15 min (12:14) | ~27 min (12:26, after a prod at 12:14) | No |
| Code complete | 35 min (12:34) | 42 min (12:41) | No |
| Test + complexity | 42 min (12:41) | 45 min (12:44) | No |

---

## Problem Statement

You are given an array `routes` representing bus routes where `routes[i]` is a bus route that the i-th bus repeats forever.

For example, if `routes[0] = [1, 5, 7]`, this means that the 0-th bus travels in the sequence `1 -> 5 -> 7 -> 1 -> 5 -> 7 -> 1 -> ...` forever.

You will start at the bus stop `source` (initially not on a bus), and you want to go to the bus stop `target`. You can travel between bus stops by buses only.

Return the least number of buses you must take to travel from `source` to `target`. Return `-1` if it is not possible.

**Example 1:**
```
Input: routes = [[1,2,7],[3,6,7]], source = 1, target = 6
Output: 2
Explanation: The best strategy is take the first bus to the bus stop 7,
             then take the second bus to the bus stop 6.
```

**Example 2:**
```
Input: routes = [[7,12],[4,5,15],[6],[15,19],[9,12,13]], source = 15, target = 12
Output: -1
```

Constraints were withheld from the statement and given on request.

---

## Conversation Log

**Interviewer [11:59]:** Presented the problem. Medium-Hard, 45-minute round, budget stated (clarify 12:03, approach 12:14, code 12:34, test 12:41). Asked if he had clarifying questions.

**Aayush:** what are the constraints ?

**Interviewer [12:00]:** `1 <= routes.length <= 500`; `1 <= routes[i].length <= 10^5`; `sum of routes[i].length <= 10^5`; all values of `routes[i]` are unique within a route; `0 <= routes[i][j] < 10^6`; `0 <= source, target < 10^6`. Asked for his approach.

**Aayush:** (silent until prodded)

**Interviewer [12:14]:** That's the approach deadline and I haven't heard anything. Talk to me — brute force, a modeling idea, anything you've got.

**Aayush:** each bus route consists of stops, and a passenger can reach any stop from any other stop within a bus route by just using 1 bus. we can use BFS on buses starting with the bus route containing the target, the BFS can give us the minimum number of buses to reach target from a stop. Initially we push the {bus number of target, 1} for all bus routes target is a part of into a queue.

**Interviewer [12:26]:** That's 27 minutes elapsed; you're 12 minutes past the approach checkpoint and the code deadline is 12:34. One question on your own approach before you code: in your BFS over buses, what exactly makes bus A a neighbour of bus B, and how are you going to compute that relation? Then write it. Do you want the `int main` boilerplate?

**Aayush:** bus A is a neighbor of bus B if the 2 buses share a common stop. We would need to maintain a set of bus routes each stop is a part of.

**Interviewer [12:27]:** Good — write it. (Provided `int main` driver carrying only the two examples from the statement.)

**Aayush:** and we keep track of visited bus and we dont push already visited bus routes into queue again

**Interviewer [12:27]:** Right. Code it — deadline 12:34.

**Aayush:** is target guaranteed to be in any route ?

**Interviewer [12:30]:** No. Neither `source` nor `target` is guaranteed to appear in any route. And they are not guaranteed to be different from each other.
<!-- HINT 1/2 — he asked only about target; the source==target observation was volunteered. -->

**Aayush [12:41]:** (submitted — see Solution section) Time complexity is O(number of different stops) - since we are visiting each stop atmost once. SC is O(routes).

**Interviewer [12:42]:** 43 minutes elapsed. Two things, quickly: (1) Dry-run your code on `routes = [[1,2,7],[3,6,7]], source = 1, target = 1`. Tell me the value it returns. (2) You said O(number of different stops) "since we are visiting each stop at most once." Justify that — in your code, how many times can a single stop be visited?

**Aayush:** need to set busesNeeded[target] = 0 at the end otherwise busesNeeded[target] = 1 and answer comes out to be 1. A stop can be visited as many times as the number of bus routes it is a part of.

**Interviewer [12:44]:** Round closed at 45 minutes.

---

## Solution

**Aayush's Final Solution (as submitted):**
```cpp
class Solution {
public:
    int numBusesToDestination(vector<vector<int>>& routes, int source, int target) {
        unordered_map<int,set<int>> stopToRoutes;
        for(int i=0;i<routes.size();i++)
        {
            for(int stop:routes[i])
            {
                stopToRoutes[stop].insert(i);
            }
        }
        unordered_map<int,int> busesNeeded;
        queue<pair<int,int>> q;
        set<int> visitedRoutes;

        // No bus route to target
        if(stopToRoutes.find(target) == stopToRoutes.end())
        {
            return -1;
        }

        for(int route: stopToRoutes[target])
        {
            q.push({route,1});
        }

        while(!q.empty())
        {
            int size = q.size();
            while(size > 0)
            {
                auto [route, dist] = q.front();
                q.pop();
                visitedRoutes.insert(route);
                for(int stop:routes[route])
                {
                    if(busesNeeded.find(stop) == busesNeeded.end())
                    {
                        busesNeeded[stop] = dist;
                    }
                    busesNeeded[stop] = min(busesNeeded[stop], dist);

                    for(int otherRoutesForStop: stopToRoutes[stop])
                    {
                        if(visitedRoutes.count(otherRoutesForStop) == 0) q.push({otherRoutesForStop, dist+1});
                    }
                }
                size--;
            }
        }

        if(busesNeeded.find(source) == busesNeeded.end()) return -1;
        return busesNeeded[source];
    }
};
```

**Defects:**
1. **`source == target` returns 1, not 0.** He found this on the named counterexample.
2. **His stated fix is incomplete and he did not notice.** "Set `busesNeeded[target] = 0` at the end" sits *after* `if(stopToRoutes.find(target) == stopToRoutes.end()) return -1;`. So `routes = [[1,2]], source = 5, target = 5` returns `-1`; correct answer is `0`.
3. **Visited marked on pop, not on push, and no skip-on-pop guard.** Between a route being pushed and popped, every other route sharing a stop can push it again; popped duplicates are fully re-expanded. Still terminates and still returns the correct minimum (FIFO with monotone `dist` saves it), but does large redundant work.
4. Minor: his `int main` quietly changed Example 2 from `(15, 12)` to `(1, 15)`; both print `-1`, so the substitution is invisible.
5. Minor: `unordered_map<int, set<int>>` adds a log factor for no benefit — values within a route are unique (a constraint he was given and didn't use), so a `vector` suffices.

**Optimal Solution:**
```cpp
int numBusesToDestination(vector<vector<int>>& routes, int source, int target) {
    if (source == target) return 0;                    // before anything else

    unordered_map<int, vector<int>> stopToRoutes;      // vector, not set
    for (int i = 0; i < routes.size(); i++)
        for (int stop : routes[i]) stopToRoutes[stop].push_back(i);

    if (!stopToRoutes.count(source) || !stopToRoutes.count(target)) return -1;

    vector<bool> visitedRoute(routes.size(), false);
    unordered_set<int> visitedStop{source};
    queue<int> q;
    for (int r : stopToRoutes[source]) { q.push(r); visitedRoute[r] = true; }

    int buses = 1;
    while (!q.empty()) {
        for (int sz = q.size(); sz > 0; sz--) {
            int r = q.front(); q.pop();
            for (int stop : routes[r]) {
                if (stop == target) return buses;
                if (!visitedStop.insert(stop).second) continue;   // stop already expanded
                for (int nr : stopToRoutes[stop])
                    if (!visitedRoute[nr]) { visitedRoute[nr] = true; q.push(nr); }
            }
        }
        buses++;
    }
    return -1;
}
```
O(S) time and space, where S = sum of route lengths.

**Time Complexity:** his answer O(number of distinct stops), "since we are visiting each stop at most once" — **wrong**; corrected under challenge to "a stop can be visited as many times as the number of bus routes it is a part of," which is closer but still understates his own code (duplicate route pops re-expand everything). · **Space Complexity:** his answer O(routes) — **wrong**; `stopToRoutes` holds one entry per (route, stop) pair, so O(S).

---

## Feedback Given

### Round conditions

**Hints used: 1/2.** When he asked "is target guaranteed to be in any route?" the answer was free — but the interviewer then volunteered "and they are not guaranteed to be different from each other," which he had not asked. That is an observation handed to him, so it counts. Ceiling: max 3/5.

**Constraints:** One open ask at 12:00, plus one genuinely good semantic question at 12:30, acted on immediately with the early `return -1`. That is the exact behaviour that has been missing for sixteen sessions. Real progress. Still never asked: whether `source` is guaranteed to be in a route (same question, other end — and he was handed the `source == target` case for free and still didn't code it), or whether a route can be a single stop.

**Complexity: volunteered unprompted.** Also new, also good — and it was wrong.

**Self-verification: partial.** No trace before submitting. Given a named input, he found the bug in one pass and stated the fix. Real improvement in trace ability, but it only happened because the input was pointed at him.

### Rubric

| Dimension | Score | Note |
|---|---|---|
| Problem understanding & clarification | 3/5 | Best clarification work in a while — one semantic question, unprompted, acted on. Undercut by not asking the mirror question about `source`. |
| Approach & thought process | 4/5 | Reached "BFS over routes, not stops" entirely unaided, with the right justification. This is the whole problem, and he got it. |
| Code quality & correctness | 2/5 | Two edge-case bugs (one caught, one not), redundant queue expansion from marking visited on pop. |
| Complexity analysis | 2/5 | Volunteered without being asked — genuinely new, credit for it. But the claim was wrong, and the correction still understated his own code. |
| Communication | 2/5 | 15 minutes of silence before the checkpoint prod, then another 12 before an approach. Once talking, clear and precise. |
| Time management | 2/5 | Approach checkpoint missed — nothing until prodded, real approach at 12:26. Code complete 12:41 vs 12:34. |

**Performance Rating: 2/5**

*This would have been a 4 — the modeling insight was unaided and correct. Two ceilings bind: one hint used (max 3), and submitted code carried a bug he never caught (max 2).*

### Algorithmic Thought-Process Debrief

**1. The derivation chain**

*Trigger — what is a node?* The problem hands you stops, so the reflex is a graph on stops with edges between consecutive stops on a route. Ask the cost: a route of length L generates L^2 stop-pairs. With `sum(L) <= 10^5` concentrated in one route, that's 10^10 edges. The stop-graph is unbuildable.

*Move — the cost function tells you the node type.* You're minimizing number of buses, not number of stops. Every edge should cost 1 unit of the thing being minimized. Riding a bus = 1. Walking between stops on the same bus = 0. When the graph has 0-cost and 1-cost edges, contract the 0-cost components — every stop on a route collapses into the route itself. He did exactly this, unaided.

*Result:* nodes = routes (<= 500), edge iff two routes share a stop, plain BFS.

*Trigger — building route-route adjacency.* Naive is 500^2 pairwise comparisons. Better: invert — `stopToRoutes[stop] = routes through it`. He got here too.

*Trigger — the last redundancy.* During BFS he expands `stopToRoutes[stop]` every time any route containing that stop is processed. After the first expansion, every route through that stop is already queued; the rest is waste.

*Move — mark stops visited, not just routes.* Two visited sets. Each route expanded once, each stop's route-list read once. O(S) total. This is the step he didn't take, and it's why his complexity answer and his actual code disagreed.

**2. The signal he missed**

Not the modeling — he nailed that. The missed signal is that **"visited" is a property of every entity you expand, not just the one you enqueue.** He wrote a BFS with two kinds of objects in it — routes (queued) and stops (traversed) — and put a visited guard on only one. That asymmetry is the whole gap between his code and O(S). It also produced defect 3: he guarded on *pop* rather than on *push*, so the guard didn't hold at the moment it needed to.

Rule: in BFS, mark visited at enqueue time, and mark every collection you expand, not just the frontier you queue.

**3. The generalization**

Class: shortest path where the graph has 0-cost and 1-cost edges, and the 0-cost edges are dense.

The tell: the quantity being minimized is not the raw entity in the input. "Fewest buses" over stops. "Fewest transfers" over stations. "Fewest words changed" over letters.

The move: contract every 0-cost component into a single node and BFS on the contracted graph. Build adjacency by inversion — map the shared attribute to the list of entities carrying it — never by pairwise comparison.

Same shape: Word Ladder II (words sharing a wildcard pattern), Minimize Malware Spread, Jump Game IV (indices sharing a value — where "clear the bucket after first expansion" is mandatory or it's O(n^2), exactly the optimization skipped here).

**4. Drills**

- **LC 1345 Jump Game IV.** Same shape, stripped down: BFS where indices sharing a value are all mutually adjacent at cost 1. Naive is O(n^2) and TLEs. The fix is precisely the one skipped tonight — after expanding the bucket for a value, clear it, because every index with that value is already queued. When it passes, write one sentence at the top of the file naming why clearing the bucket was necessary, then apply that sentence back to this problem.
- **Process drill:** the approach was correct at 12:26 and pieces of it existed well before. 15 minutes of silence cost the round. Hard rule: at 5 minutes, say the brute force out loud, even if it's terrible. The first idea is not what's graded; the interviewer knowing what you're thinking is.
