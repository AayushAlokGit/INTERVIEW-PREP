# DSA Round Transcript
**Date:** 2026-06-22
**Start Time:** 9:02
**End Time:** 9:55
**Duration:** 53 minutes
**Problem:** Reconstruct Itinerary
**Topic:** Graphs (Eulerian Path / Hierholzer's Algorithm), DFS
**Difficulty:** Hard

---

## Problem Statement
You are given a list of airline tickets where `tickets[i] = [from_i, to_i]` represents the departure and arrival airports of one flight. Reconstruct the itinerary in order and return it. All tickets belong to a person who departs from `"JFK"`, so the itinerary must begin with `"JFK"`.

Rules:
- Must use all tickets exactly once each.
- If multiple valid itineraries exist, return the one with the smallest lexical order (read as a single string of airports).
- At least one valid itinerary is guaranteed.

Constraints:
- `1 <= tickets.length <= 300`
- Airport codes are 3 uppercase letters; `from_i != to_i`.
- Duplicate `[from, to]` tickets are allowed.

**Example 1:**
```
Input:  tickets = [["MUC","LHR"],["JFK","MUC"],["SFO","SJC"],["LHR","SFO"]]
Output: ["JFK","MUC","LHR","SFO","SJC"]
```

**Example 2:**
```
Input:  tickets = [["JFK","SFO"],["JFK","ATL"],["SFO","ATL"],["ATL","JFK"],["ATL","SFO"]]
Output: ["JFK","ATL","JFK","SFO","ATL","SFO"]
```

---

## Conversation Log

**Interviewer:** Presented the problem with examples and constraints. Asked for clarifying questions before approach.

**Aayush:** Noted start time 9:02. Asked: what are the constraints?

**Interviewer:** Gave constraints (length ≤ 300, 3-letter codes, no self-loops, duplicates allowed, valid itinerary guaranteed).

**Aayush:** If there are two identical flights, do we output only one of them in the itinerary?

**Interviewer:** No — you must use every ticket exactly once, so both identical tickets must appear; the itinerary always has N+1 airports for N tickets.

**Aayush:** Could you give an example to explain this?

**Interviewer:** Gave `[["JFK","ABC"],["JFK","ABC"],["ABC","JFK"]]` → `["JFK","ABC","JFK","ABC"]`, walking through which ticket each hop uses.

**Aayush:** Proposed modeling as `map<string, map<string, vector<int>>>` where `graph[u][v]` = list of ticket indices for tickets `{u,v}`. Start DFS from JFK; explore one part fully before returning to current node and exploring others. A neighbor v is available if `graph[u][v].size() > 0`; pop a ticket when used so it can't be reused. Ordered map keeps neighbors sorted.

**Interviewer:** Solid modeling. Pushed on the DFS subtlety with `[["JFK","KUL"],["JFK","NRT"],["NRT","JFK"]]` — asked how exactly the output list is built and where an airport gets added.

**Aayush:** DFS would stop at KUL (no edge back to JFK), so we need to explore reversible branches first. JFK is the root; only one subtree may have only outgoing edges, the others must be fully reversible. So explore neighbors with back-edges first.

**Interviewer:** Noted he was trying to pre-decide subtree order, which is messy. Gave the reframe hint: always go greedily to the smallest neighbor; the trick is *when* you add a node — once a node has no edges left it's "finished" and must go at the end. Asked: add on arrival or after exhausting all outgoing edges, and what to do about the order?

**Aayush:** Push the node into the result after exploring its neighbours, and reverse the result.

**Interviewer:** Confirmed — that's Hierholzer's (post-order add + reverse). Traced the KUL example to validate. Asked him to write the full code.

**Aayush:** Submitted C++ solution (below). Stated TC = O(V+E), SC = O(V).

**Interviewer:** Verified code is correct on both cases. Challenged complexity: `std::map` is a BST, not a hash map — what's the construction and lookup cost? And space: graph stores every ticket index, plus recursion depth.

**Aayush:** Corrected: TC = O(E log V) (E tickets, log V for neighbor lookup via ordered map). SC = O(E) for recursion stack worst case and O(E) for graph storage.

**Interviewer:** Both correct. Asked optimization questions: (1) can the log be removed? (2) is `vector<int>` of indices needed?

**Aayush:** (1) Use unordered_map for O(1) lookups. (2) Store a count instead of a vector; claimed this also reduces SC to O(V).

**Interviewer:** Two catches — unordered_map breaks the lexical-order guarantee, and SC isn't O(V). Asked him to resolve both.

**Aayush:** (2) SC remains bounded by O(E). (1) Sort the neighbours before iterating.

**Interviewer:** Correct — sorting neighbors with unordered_map gives O(E log E) total, so the log can't be eliminated when lexical order is required; the std::map solution is already at the optimal complexity class. Moved to feedback.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

void dfs(map<string, map<string, vector<int>>> &g, string node, vector<string> &res)
{
    for (auto &[nbr, ticketIndices] : g[node])
    {
        while (!ticketIndices.empty())
        {
            ticketIndices.pop_back();
            dfs(g, nbr, res);
        }
    }
    res.push_back(node);
}

int main() {
    vector<vector<string>> tickets{{"JFK","ABC"},{"JFK","ABC"},{"ABC","JFK"}};
    map<string,map<string,vector<int>>> g;

    for(int i=0;i<tickets.size();i++)
        g[tickets[i][0]][tickets[i][1]].push_back(i);

    vector<string> res;
    dfs(g,"JFK",res);
    reverse(res.begin(),res.end());

    for(string s:res) cout<<s<<" ";
    return 0;
}
```

**Optimal Solution (same complexity class):** The submitted solution is optimal. A minor refinement: replace `vector<int>` with an `int` count (or `multiset`), since the index values are never read.

**Time Complexity:** O(E log V) — E = number of tickets; log V per ordered-map neighbor lookup. (Cannot go below O(E log E) because lexical ordering requires sorting.)
**Space Complexity:** O(E) — graph storage (worst case all distinct edges) + recursion stack depth on a long chain + result list.

---

## Feedback Given

**Strengths**
- Excellent graph modeling instinct — landed on `map<string,map<string,vector<int>>>` and immediately saw the ordered map gives lexical ordering for free.
- Strong intuition about the dead-end structure (one branch has only outgoing edges, others must be reversible).
- Clean, correct, elegant final code — the shared-reference trick (outer `while` sees the vector emptied by the recursive call) makes Hierholzer work without extra bookkeeping.
- Responsive to hints and self-corrected well on complexity once challenged.

**Areas to improve**
1. **Complexity was wrong on the first pass for BOTH time and space.** You asserted O(V+E) / O(V) — missing the `log V` from the ordered map, and undercounting space (graph is O(E), recursion is O(E)). This is a recurring pattern. Make it a habit: before stating complexity, ask "what container am I using and what does each operation actually cost?" — `std::map` is never O(1).
2. **Over-engineered the approach before the key insight.** You spent time trying to pre-decide which subtree to visit first (which neighbors have back-edges). The elegant insight — post-order add + reverse — sidesteps all of that. When an ordering decision feels complicated, consider whether deferring the decision (post-order, processing-on-exit) dissolves it.
3. **Optimization missed a problem constraint.** Your first optimization (unordered_map for O(1)) silently dropped the lexical-ordering requirement. When optimizing, re-check every constraint the original design was satisfying before declaring the win.
4. **Didn't dry-run your own code.** You declared it done and stated complexity without tracing it yourself — I verified correctness for you. Build the habit of a quick self-trace on a small case before handing it over.

**Scorecard**
| Category | Score | Notes |
|---|---|---|
| Problem Understanding & Clarification | 4.0 / 5 | Asked about constraints and duplicate semantics; took a couple rounds to internalize duplicates but got there. |
| Approach & Thought Process | 3.5 / 5 | Great modeling and intuition, but went down a complicated pre-ordering path; needed a hint for the post-order insight. |
| Code Quality & Correctness | 4.5 / 5 | Clean, correct, elegant. Lost half a point for not self-verifying. |
| Complexity Analysis | 2.5 / 5 | Wrong on both TC and SC initially; corrected only after prompting. The weakest area today. |
| Communication | 3.5 / 5 | Clear and engaged, corrected well under pushback; missed a constraint during optimization. |

**Overall: 3.6 / 5** — Reached the optimal solution with one structural hint. Biggest lever for next time: nail complexity on the first try by reasoning from your actual data structures, and dry-run your own code before declaring done.

**Time Taken: 53 minutes**
