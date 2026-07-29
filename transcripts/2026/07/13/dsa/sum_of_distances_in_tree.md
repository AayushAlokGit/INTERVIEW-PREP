# DSA Round Transcript
**Date:** 2026-07-13
**Start Time:** 16:26
**End Time:** 16:51
**Duration:** 25 minutes
**Problem:** Sum of Distances in Tree
**Topic:** Trees / Graphs — Rerooting (Two-Pass DFS)
**Difficulty:** Hard

**Outcome:** Not solved — candidate gave up before writing any code.

---

## Problem Statement

There is an undirected **connected tree** with `n` nodes labeled `0` to `n - 1` and `n - 1` edges.

You are given the integer `n` and the array `edges`, where `edges[i] = [ai, bi]` indicates that there is an edge between nodes `ai` and `bi` in the tree.

Return an array `answer` of length `n`, where `answer[i]` is the **sum of the distances between node `i` and all other nodes** in the tree.

(Distance between two nodes = number of edges on the path between them.)

**Example 1:**
```
Input:  n = 6, edges = [[0,1],[0,2],[2,3],[2,4],[2,5]]
Output: [8,12,6,10,10,10]
```
Tree:
```
        0
       / \
      1   2
         /|\
        3 4 5
```
- `answer[0] = 8`: dist(0,1)=1, dist(0,2)=1, dist(0,3)=2, dist(0,4)=2, dist(0,5)=2 → 1+1+2+2+2 = 8
- `answer[2] = 6`: dist(2,0)=1, dist(2,1)=2, dist(2,3)=1, dist(2,4)=1, dist(2,5)=1 → 6

**Example 2:**
```
Input:  n = 1, edges = []
Output: [0]
```

**Example 3:**
```
Input:  n = 2, edges = [[1,0]]
Output: [1,1]
```

**Constraints (given on request):**
- `1 <= n <= 30,000`
- `edges.length == n - 1`
- `0 <= ai, bi < n`
- `ai != bi`
- Input is guaranteed to be a valid connected tree.

---

## Conversation Log

**Interviewer:** [Presents problem.] Note the current time. Any clarifying questions?

**Aayush:** 16:26

**Interviewer:** Start time recorded. Clarifying questions, or straight to your read on the problem?

**Aayush:** what are the constraints?

**Interviewer:** `1 <= n <= 30,000`, `edges.length == n-1`, `0 <= ai, bi < n`, `ai != bi`, and the input is guaranteed to be a valid connected tree. No cycles, no disconnected components, no self-loops.

**Aayush:** we can get all pairs shortest paths using floyd warshall in n^3 and then get the answer in n^2 brute force

**Interviewer:** Floyd-Warshall gives you all-pairs on a general *weighted* graph. Look at what you actually have: an unweighted *tree*. Two questions: (1) Given a single source node `s`, what's the cheapest way to get distances from `s` to every other node in a tree? (2) So what does that make the honest brute force, and is it good enough for n = 30,000?

**Aayush:** for each node simply start BFS from that node and we will get the distances to all nodes upon BFS completion. O(V + E = n) — BFS, and for each index O(n). So brute force becomes O(n^2)

**Interviewer:** Correct — BFS from a source is O(n) in a tree, so brute force is O(n²) = 9×10⁸. Too slow for n = 30,000. Those n BFS traversals compute enormously overlapping information — just like the n simulations in the last problem. Think about a parent `u` and one of its children `v`, connected by a single edge. Suppose I hand you `answer[u]` for free. Can you compute `answer[v]` from it — and what extra information would you need to do that in O(1)?

**Aayush:** what does connected tree mean here?

**Interviewer:** Every node is reachable from every other. Combined with exactly `n-1` edges, that forces a tree — no cycles. Properties you can rely on: exactly one simple path between any two nodes (so distance is unambiguous); it's given undirected and unrooted, but you're free to *pick* a root (say node 0) for your traversal; and removing any single edge `(u,v)` splits the tree into exactly two components. That last one is worth sitting with. Now go back to my question: parent `u`, child `v`, I give you `answer[u]` free — what is `answer[v]`?

**Aayush:** this is a spanning tree

**Interviewer:** Not really — a spanning tree is a subgraph extracted from some larger graph. There's no larger graph here; this just *is* a tree. You're circling terminology instead of engaging the question.

Take Example 1 rooted at 0. `answer[0] = 8`, `answer[2] = 6`. Consider edge `(0,2)`. Deleting it splits the tree into:
- Piece containing 2: `{2,3,4,5}` — 4 nodes.
- Piece containing 0: `{0,1}` — 2 nodes.

Now compare `answer[0]` and `answer[2]` node by node. When you shift viewpoint from 0 to 2: for every node in `{2,3,4,5}`, did its distance go up or down, and by how much? For every node in `{0,1}`, same question? Answer those, then give me the formula relating `answer[2]` to `answer[0]`.

**Aayush:** give me a hint

**Interviewer:** No. That *is* the hint. I've already split the tree and counted both sides. What's left is arithmetic on a 6-node example you can do on paper in sixty seconds. This is the exact reflex that cost you the last round — you stop one step before the answer and ask for a push. In a real interview the interviewer stops pushing.

Answer just this one question: You're standing at node 0. Your distance to node 3 is 2. Now you take one step and stand at node 2. What is your distance to node 3 now? Up or down, by how much? Then the same for node 1 (distance from 0 is 1 → distance from 2 is ?).

**Aayush:** give me the answer

**Interviewer:** I'll give you this one, but understand this is the second time this round and the third across two rounds that you've asked me to hand you a step you could have done on paper. That's a scoring event.

Standing at 0, you move one step to 2:
- Node 3 was at distance 2. Now it's 1 — **closer by 1**. Same for 4, 5, and 2 itself. Every node in `{2,3,4,5}` got 1 closer, because your step moved you *toward* all of them.
- Node 1 was at distance 1. Now it's 2 — **farther by 1**. Same for 0 itself. Every node in `{0,1}` got 1 farther.

One step along an edge changes every node's distance by exactly ±1, and the sign depends only on which side of that edge the node lives.

So let `cnt[v]` = number of nodes in `v`'s subtree (the side containing `v` when you cut edge `(u,v)`). Then:
```
answer[v] = answer[u] - cnt[v] + (n - cnt[v])
```
Check: `answer[2] = 8 - 4 + (6-4) = 6`. ✓

Now you do the rest. That gives every node's answer in O(1) from its parent — but only if you have `cnt[]` for every node and `answer[root]` for one starting node.
1. How do you compute `cnt[v]` for all `v`? What traversal, in what order?
2. How do you compute `answer[0]` — in the *same* traversal, without a separate pass?
3. How do you propagate to everyone else?

**Aayush:** lets park this question and we will return to it later

**Interviewer:** Parking it *is* the weakness. "Stops at 'not sure' / abandons correct idea on obstacle" is the single most recurring communication failure in your file, and it just fired three times in fifteen minutes. And you are *not stuck* — the hard part is done. What's left is three mechanical steps you've written before: (1) post-order DFS for subtree sizes; (2) accumulate `answer[0]` in that same DFS; (3) pre-order DFS applying the reroot formula. Two passes, O(n).

If you walk away here, what you've trained today is the walking away. Your call: (a) give me the three steps + code, ~15 minutes, or (b) stop and I close out with feedback.

**Aayush:** i give up

**Interviewer:** Understood. Note the current time. [Reveals full solution.]

**Aayush:** 16:51

---

## Solution

**Aayush's Final Solution:** *None — no code was written.*

Progress reached: identified the O(n²) brute force (BFS from every source). Did not derive the rerooting relation; it was revealed by the interviewer after three requests. Did not implement.

**Optimal Solution (revealed):**
```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution {
    vector<vector<int>> adj;
    vector<int> cnt, ans;   // cnt[v] = size of v's subtree; ans[v] = sum of distances

    // Pass 1 (post-order): compute subtree sizes, and build ans[root] as a by-product.
    void dfs1(int u, int parent) {
        cnt[u] = 1;
        for (int v : adj[u]) {
            if (v == parent) continue;
            dfs1(v, u);
            cnt[u] += cnt[v];
            ans[u] += ans[v] + cnt[v];   // each of cnt[v] nodes is 1 edge farther via u
        }
    }

    // Pass 2 (pre-order): reroot from parent to child in O(1)
    void dfs2(int u, int parent, int n) {
        for (int v : adj[u]) {
            if (v == parent) continue;
            ans[v] = ans[u] - cnt[v] + (n - cnt[v]);
            dfs2(v, u, n);
        }
    }

public:
    vector<int> sumOfDistancesInTree(int n, vector<vector<int>>& edges) {
        adj.assign(n, {});
        cnt.assign(n, 0);
        ans.assign(n, 0);

        for (auto &e : edges) {
            adj[e[0]].push_back(e[1]);
            adj[e[1]].push_back(e[0]);
        }

        dfs1(0, -1);        // ans[0] correct; cnt[] correct for all
        dfs2(0, -1, n);     // propagate to everyone else
        return ans;
    }
};
```

**Key derivation (the two pieces Aayush did not reach):**

*Reroot formula.* Stepping from `u` to an adjacent child `v` moves you 1 closer to every node on `v`'s side (`cnt[v]` of them) and 1 farther from every node on `u`'s side (`n - cnt[v]` of them):
```
answer[v] = answer[u] - cnt[v] + (n - cnt[v])
```

*Root's total, free in the same DFS.* After `dfs1(v, u)` returns, `ans[v]` is the sum of distances within `v`'s subtree measured from `v`. Re-measuring those same `cnt[v]` nodes from `u` makes each exactly one edge farther, so `ans[u] += ans[v] + cnt[v]`. Equivalently `ans[root] = Σ depth(x) = Σ_{v ≠ root} cnt[v]`, since each node is counted once per ancestor edge above it.

**Time Complexity:** O(n) — two DFS passes, each touching every node and edge once. (Aayush reached only the O(n²) brute-force bound, which he stated correctly.)

**Space Complexity:** O(n) — adjacency list + `cnt` + `ans` + recursion stack. Note the recursion stack is O(n) in the worst case (path-shaped tree), so an iterative/explicit-stack version is the safe move at n = 30,000. (Not discussed — round ended first.)

---

## Feedback Given

# Feedback — Sum of Distances in Tree (Hard)

## Overall: 2.0 / 5 — No Hire

I'm going to be blunt, because a soft version of this feedback would be useless to you. **You did not fail this problem on ability. You failed it on persistence.** Ninety minutes earlier you independently cracked Odd Even Jump — a strictly harder problem — and the *core insight there was the same shape as the one here*: "don't redo n overlapping traversals, exploit what the previous one already told you." You had the pattern in your hands and you put it down.

---

### Problem understanding & clarification — 2.5/5
You asked for constraints — good, that reflex is holding. But then you spent two turns on **"what does connected tree mean"** and **"this is a spanning tree"**, which is vocabulary anxiety, not clarification. A connected tree is the single most standard input in graph interviews; you've solved Course Schedule, Alien Dictionary, Word Ladder, and Serialize/Deserialize Binary Tree. You know what a tree is.

What that told me as an interviewer is that you'd hit friction and were reaching for *anything* to say other than engaging with the geometry of the problem. Retreating into definitional questions when you're stuck reads as stalling. It is better to say "I'm stuck, here's what I've tried, here's where my reasoning breaks" — that's a signal of a strong candidate. Asking me to re-explain "connected" is not.

### Approach & thought process — 2.0/5
Opening with **Floyd–Warshall** was a bad instinct and worth examining. Floyd–Warshall is O(V³) all-pairs on a general *weighted* graph. You had an *unweighted tree*. Reaching for the heaviest general-purpose hammer when the input has enormous exploitable structure is precisely the entry in your file that reads *"Defaults to generic pattern over structure-exploiting one"* — 15 sessions now, and this is the cleanest example of it yet. The correct first thought on seeing a tree is "trees are almost never a general-graph problem; what does the tree structure buy me?"

You recovered to O(n²) BFS quickly and correctly once nudged — that part was fine.

Then you stopped. I gave you a split tree, both component sizes counted, a worked example, and a single arithmetic question: *"you're at node 0, distance to node 3 is 2; you step to node 2; what's the distance now?"* The answer is 1. That is the entire problem. You did not attempt it.

### Code quality & correctness — N/A
**No code was written.** In a real loop this is an automatic reject regardless of everything else — there is no artifact to evaluate.

### Complexity analysis — 3.0/5
The one bright spot. "BFS is O(V + E) = O(n) in a tree, done from n sources, so O(n²)" was crisp, correct, and immediate. No hedging, no missing factors. When you're on solid ground your complexity reasoning is genuinely fine, which reinforces that today's problem was not analytical.

### Communication — 1.5/5
Three escalating bails in fifteen minutes:
1. *"give me a hint"* — after I'd already given a hint.
2. *"give me the answer"* — after I'd narrowed it to one-step arithmetic on a 6-node example.
3. *"lets park this"* → *"i give up"* — with the hard insight already handed to you and only mechanical work remaining.

That's the tell. You didn't quit at the hard part. **You quit at the easy part, *after* the hard part was already solved for you.** Steps 1–3 that remained — subtree sizes via post-order DFS, accumulate the root's total, propagate top-down — are things you have unambiguously written before.

---

## What actually happened here, and what to do about it

This wasn't a knowledge gap. It was **friction tolerance**. You'd been going for 97 minutes on a hard problem, you were depleted, you hit a moment of not-immediately-seeing-it, and your reflex under load is to look for an exit. That reflex is the single highest-leverage thing you can fix, and it is worth more to your interview outcomes than another twenty problems' worth of pattern memorization.

The uncomfortable truth: interviewers do not primarily score whether you got it. They score **what you do when you don't have it yet.** A candidate who says "I don't see it — let me try the smallest concrete case and see what falls out" and then grinds forward on a 6-node example gets hired far more often than one who solves it fast but folds the moment they're uncertain. You have the first behavior in you (you showed it all through Odd Even Jump). You just don't have it *when tired*.

**Three concrete rules, starting next session:**

1. **Ban the phrase "give me a hint" until you have written something down.** Anything. A brute force, a 3-node example, a wrong formula. Earn the hint by producing an attempt first.
2. **When stuck on a tree/graph/array problem, shrink the input to n = 3 or n = 4 and compute the answer by hand.** Nearly every structural insight — including today's — is visible in the smallest non-trivial case. You never once did this today.
3. **Cap your sessions.** You went 97 minutes, then straight into a Hard. The give-up here is at least partly a fatigue artifact, and fatigue-quitting is a habit you can accidentally train. Two hard problems back-to-back with no break is not a rep, it's attrition.

**Time Taken: 25 minutes**
