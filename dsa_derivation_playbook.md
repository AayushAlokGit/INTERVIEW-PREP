# DSA Derivation Playbook

Induced bottom-up from 107 solved-problem rounds (2026-04-29 → 2026-07-28) in
`transcripts/<Y>/<M>/<D>/dsa/`. Organised by *what you notice* → *what you do*, not by LeetCode tag.

**Read order under time pressure:** §5 card → §4 process → §1 moves. Everything else is why.

---

## 0. Method, and what to distrust

107 rounds, 104 with a usable rating (mean 3.68, 66% in the 3.5–4.4 band). The rating scheme
changed four times across the corpus; all normalised to /5.

Overall rating is **not** the right lens. The 5-criterion average is dragged up by Complexity and
Communication scores that stay high even when the algorithm never arrived — *Minimum Cost to Hire
K Workers* scores 3.4 overall but 2/5 on Approach. And it is dragged down by rounds where the
algorithm arrived in six minutes and a bug did the damage (*Maximum Score of a Good Subarray*,
3.0). So §3 uses **derivation failure** — the pivotal move had to be hinted, handed over, or never
arrived — which is 26 distinct problems across 29 rounds.

Three problems have no rating (`coin_change`, `cheapest_flights_within_k_stops`,
`minimum_cost_to_reach_destination_in_time`) and were classified from content. Three files are the
same problem — 132 Pattern, attempted 07/22, 07/27, 07/28, scoring 1 → 2 → 4. They are counted as
three rounds because they are three outcomes, and that progression is the best evidence in the
corpus that this is learnable.

**The taxonomy was tested, and it broke once.** Re-classifying all 107 problems produced 21
dual-fit collisions, and **19 of them involved the same category** — "run it in the order that
makes the unknown already known." That is not ambiguity, it is a category error: order is a
decision you make *after* choosing a move. It was demoted to the modifier **D** and its members
re-homed, which is why M3 is missing from the numbering. One split was also needed: LRU Cache,
Median from Data Stream and Max Frequency Stack share an entry point the others don't (an API with
per-operation targets), so they became **M14**. After that, 3 of 107 remain genuinely dual (2.8%)
and exactly one problem fits nothing — **String to Integer (atoi)**, where the brute force *is* the
optimal and the whole difficulty is enumerating the spec. Left unclassified deliberately.

**Where the evidence is thin:** only 9 transcripts carry a formal derivation debrief (07/23
onward); those nine are far richer than the other 98 and are over-represented in the trigger
wording below. M11 (4 problems), M12 (4), M14 (3) are thinly evidenced — M12's 50% failure rate
rests on two rounds, so do not weight it against M6, which rests on six. Below M2, the failure-rate
denominators are small enough that the gap between M8 at 29% and M1 at 21% is noise. The
derivation-failure reclassification is a judgement call on ~30 rounds, not a number lifted from the
transcripts.

---

## 1. The moves

Twelve moves, one modifier, one meta-move. Named as **actions**, each fired by an observation you
can make from the statement or from staring at your own brute force. Counts and failure rates:

| Move | n | Failure rate |
|---|---:|---:|
| M7 — Resize the state until it is exactly what the future needs | 18 | 17% |
| M1 — Fix the element that decouples the rest | 14 | 21% |
| M5 — Carry the inner loop's answer forward | 12 | **8%** ← strongest |
| M2 — Search the answer's value instead of constructing it | 11 | **45%** |
| M6 — Commit optimistically, then retract | 9 | **67%** ← worst |
| M10 — Turn the relation into a graph, name the traversal | 9 | 22% |
| M4 — Re-index by the thing you're querying | 8 | 38% |
| M8 — Sort by the factor you want to freeze | 7 | 29% |
| M9 — Rewrite the objective into an equivalent one | 7 | 29% |
| M11 — One pass per direction / carry both extremes | 4 | 0% |
| M12 — Compute one answer, reach the neighbour by a delta | 4 | 50% |
| M14 — Compose structures to hit per-operation targets | 3 | 0% |
| unclassified — String to Integer (atoi) | 1 | — |

Cross-cutting: **D — pick the direction/order** (~35 problems, always a *second* decision).
Meta: **M13 — spend the constraint** (runs at minute 2, selects among the rest).

Ordered below by how badly they beat you, not by size.

---

### M13 — Spend the constraint · meta-move, run at minute 2

Not a peer of the others. It produces no algorithm; it tells you **which one to look for**. It is
the single most frequently named miss in the corpus.

| You were told | Do this |
|---|---|
| A target complexity tighter than your brute force | Ask what the bound **forbids**. `O(n)` kills sorting and heaps |
| A tiny bounded universe (26, 32, 10, 9, 100) | Put a loop around it, or index an array by it |
| `n ≤ 300` / `n ≤ 500`, DP-shaped | Cubic intended → interval DP, 2D state + 1D transition |
| `n ≤ 20` | Exponential intended → bitmask over subsets |
| Bounded value range (`< 10^5`) | Counting sort / bucket / Fenwick over the value axis |
| Values can be **negative** | Sliding window is dead → prefix sums + hash, or a deque |
| Values are all **positive** | Monotonicity available → two pointers |
| Input sorted, or rows/cols sorted | Binary search or staircase walk is free |
| "at most k" on a sequence of choices | Greedy unsafe by default; k joins the state |
| "both play optimally" + fixed total | `dp[i] = max(gain − dp[next])`, single value, sign flip |
| A quantity no datatype holds | An **approach** problem, not a datatype problem |
| Can I mutate the input? *(you must ask)* | If your aux structure ⊆ already-consumed input, store it there |

Evidence it is real and repeatedly missed: Longest Consecutive Sequence (*"the O(n) requirement IS
the hint"*), Burst Balloons (`n ≤ 300` read past — it specifies cubic), Longest Substring with At
Least K Repeating (26 read past for 23 minutes), Odd Even Jump (asked for constraints at minute
two, never spent them), Minimum Cost to Cut a Stick (`n ≤ 10^6` not connected to an `n×n`
allocation).

---

### M6 — Commit optimistically, then retract · 9 problems · ⚠ 67% failure, your worst

| Trigger | Do this |
|---|---|
| You only learn whether a choice was right **after** you've passed it | Take everything, keep an undo structure |
| Two resource types, one strictly better but scarcer | Spend the scarce one greedily, demote the worst use later |
| "Maximise the count under a cumulative budget", forced arrival order | Take-then-regret max-heap |
| Building a sequence left-to-right under a lexicographic objective | Monotonic stack, pop while worse |
| A greedy that *almost* works but breaks on a counterexample | Don't abandon greed — add an **undo** |
| Elements interact only with their nearest surviving neighbour | Stack of survivors |

**Diagnostic when a greedy fails:** *is my candidate set too small, or is my move set too small?*
Coverage failure and expressiveness failure have opposite fixes. If it's the move set → add `pop`.

**The pop guard is always a feasibility question**, and it is problem-specific: "does this letter
occur later?", "do I have budget left?", "is the retraction still legal?" Getting the guard right
*is* the correctness argument.

Members: Min Refueling Stops · Course Schedule III · Furthest Building · Min Intervals to Cover
Target · Max Events · Maximum Width Ramp · Asteroid Collision · Remove Duplicate Letters · Basic
Calculator II

**Fails when:** retraction isn't legal — an earlier commitment already changed the world
irreversibly. Then you need real DP (M7). *"At most k transactions"* + non-overlap (Best Time IV)
is the canonical trap.

---

### M2 — Search the answer's value instead of constructing it · 11 problems · ⚠ 45% failure

| Trigger | Do this |
|---|---|
| "minimise the maximum" / "maximise the minimum" | Binary search the answer + feasibility check |
| "kth smallest/largest" where counting `≤ v` is cheap | Binary search value space, monotone `count(v)` |
| "smallest value such that X is possible" | Same |
| Producing all candidates is huge; testing one is easy | Same |
| The candidate space is small and bounded | **Enumerate** it instead of binary-searching |
| The answer is provably a contiguous window | Search for its boundary, not its contents |

**Template, always:** `while(lo<hi){ if(ok(mid)) hi=mid; else lo=mid+1; } return lo`. Never a
three-way `== target` search — *"'count == k' is the wrong anchor; you want the smallest d with
count(d) ≥ k."*
**Anchor rule:** the comparison anchor must be a fixed end (`nums[hi]`), never a moving one.

Members: Split Array Largest Sum · Kth Smallest in Sorted Matrix · Median of Two Sorted Arrays ·
Min Days to Make M Bouquets · Kth Smallest Pair Distance · Find K Closest · Count Complete Tree
Nodes · Search in Rotated Sorted Array · First & Last Position · Find Min in Rotated II · Longest
Substring with At Least K Repeating

**Fails when:** `ok()` isn't monotone, or the predicate is uninformative on ties (duplicates in
Find Min in Rotated II — handle separately and expect to lose the log bound).

---

### M12 — Compute one answer, reach the neighbour by a delta · 4 problems · ⚠ 50% failure (thin)

| Trigger | Do this |
|---|---|
| "for **every** node/cell/update, report the answer" and one instance is O(n) | Don't run it n times |
| Adjacent positions differ predictably | Find the **O(1) delta** and propagate |
| A query for every candidate modification | Precompute the static structure once; each query is bounded |
| Online additive updates to a static problem | Maintain incrementally (DSU + running count) |

Members: Sum of Distances in Tree · Making a Large Island · Number of Islands II · Partition Labels

**Fails when:** the update isn't local — a change that perturbs the whole structure has no delta.

---

### M7 — Resize the state until it is exactly what the future needs · 18 problems · largest

| Trigger | Do this |
|---|---|
| Two arrivals at the same place lead to different futures | **Add** that difference to the state |
| A budget / count / parity / held item a post-check can't fix | Add it |
| A constraint that can make a *longer* answer better | It belongs in the state, not a post-check |
| Shortest path **plus** a secondary budget | Dijkstra's dominance is dead → level-relax or augment |
| You wrote a param and can't say "if it changes, my best move changes" | **Drop** it — it's history |
| Two indices always within a bounded distance | Collapse to one |
| Two agents advancing in lockstep | One coordinate is redundant |
| The answer depends on **how you got here** | Memo is *invalid*; only pruning is available |
| A DP param with huge numeric range but few real values | Index by the value list, not the range |

**The audit, one line per parameter:** *"does changing it change my best move? yes → state; no →
not state."*
**DP branch rule:** *for this branch, what did I consume from each input? Subtract exactly that.*
**Branch-identity check:** two semantically different branches with identical formulas = a bug,
every time.

Members (add): Validate BST · Shortest Path w/ Obstacles · Cheapest Flights K Stops · Min Cost in
Time · Best Time IV · Restore IP · Word Search · RegEx Matching · Dungeon Game · Longest Increasing
Path
(drop/collapse): Decode Ways · Cherry Pickup II · Stone Game III · Word Break · Edit Distance ·
Coin Change · Palindrome Partitioning · Combination Sum

**Fails when:** the state is genuinely path-dependent (Word Search) — no memo exists. Also when a
parameter's numeric range is huge but its real domain is small (Cut a Stick sized the memo `n×n` =
10^12 when the domain was the cut list).

---

### M1 — Fix the element that decouples the rest · 14 problems

| Trigger | Do this |
|---|---|
| Nested loops over a subrange, value determined by **one** element (min/max/peak) | Enumerate that element; ask what range it owns |
| A constraint over 3+ indices (`i<j<k` with value relations) | Fix the index that **disconnects** the constraint graph |
| Sum/count over all O(n²) subranges | Flip to per-element contribution |
| 2D "count all submatrices" | Fix two of four indices → reduces to the 1D version |
| A symmetric property (palindrome) | Enumerate the axis, not the endpoints |
| Removing an element changes who is adjacent to whom | Fix the **LAST** operation, not the first |
| Splitting on the first move leaves the halves coupled | Same — fix the last |

**The test:** after fixing an index, write the residual query as a literal sentence and count its
dimensions. **Still 2-D? You fixed the wrong index.**
**The tell:** the moment you say *"for each element I need the nearest/largest/smallest in one
direction satisfying an inequality"* — that is a monotonic stack, every time.

Members: Largest Rectangle · Sum of Subarray Minimums · Maximal Square · Longest Valid Parentheses
· Longest Palindromic Substring · Count of Range Sum · Min Cost to Cut a Stick · Burst Balloons ·
Number of Submatrices Sum to Target · 132 Pattern ×3 · Insert Interval · Next Permutation

**Fails when:** the objective isn't determined by a single element (sum-based subarray problems →
M4 or M5).

---

### M5 — Carry the inner loop's answer forward · 12 problems · your strongest, 8% failure

| Trigger | Do this |
|---|---|
| O(n·k) or O(n²) loop whose inner part is a range aggregate over a monotonically shifting set | Deque / heap / running accumulator |
| "longest/shortest contiguous run under a constraint that only gets easier when you remove elements" | Two-pointer window |
| Range max/min over every fixed-size window | Monotone deque |
| Repeated order-statistic queries where only **one position** is ever asked | Two heaps / size-k heap |
| Inner loop = "best previous state satisfying an inequality", states ordered by DP value | Binary search the scan away |
| Transitions look like "choose among many" but the rule picks exactly one | Precompute the successor; the search collapses to a chain |
| The validity check is itself O(alphabet) | Find a scalar summary that updates in O(1) |
| A max in the window that's expensive to maintain | Ask whether a stale max is **harmless** (monotone answer) |

**Precondition, run before writing any window:** *"if [l,r] fails, must every window containing it
fail?"* If no → M2 (enumerate a bounded cap) or prefix sums + deque.

Members: LS At Most K Distinct · Min Window Substring · Longest Repeating Char Replacement ·
Subarray Product < K · Sliding Window Maximum · Jump Game VI · Shortest Subarray Sum ≥ K · Jump
Game II · Kth Largest · Smallest Range K Lists · LIS · Odd Even Jump

**Fails when:** the predicate isn't monotone (Longest Substring with At Least K Repeating kills the
window outright), or negatives are allowed (→ prefix + deque, or M4).

---

### M10 — Turn the relation into a graph, name the traversal · 9 problems

| Trigger | Do this |
|---|---|
| Pairwise relations that **compose transitively** (ratios, rates, offsets) | Weighted graph, path aggregate |
| Recover a total order from local pairwise evidence | DAG + topological sort |
| Something **spreading** from many origins at once | Multi-source BFS, levels = time |
| Shortest transformation / fewest steps, unweighted | BFS |
| "min over paths of the max on the path" | Dijkstra with a min-heap frontier |
| Connect everything at minimum total cost, no ordering required | MST (check density: dense → array Prim) |
| Use every edge exactly once | Eulerian path (Hierholzer: post-order + reverse) |
| Online connectivity updates | Union-Find |
| Values are valid indices into their own array | Functional graph → cycle detection |

**Anti-trigger:** the input is a **tree**. A tree is almost never a general-graph problem — Sum of
Distances in Tree opened with Floyd–Warshall on an unweighted tree, the cleanest instance of
reaching for the heaviest general hammer in the corpus.

Members: Evaluate Division · Alien Dictionary · Rotting Oranges · Trapping Rain Water II · Min Cost
to Connect All Points · Find the Duplicate Number · Pacific Atlantic · Reconstruct Itinerary · Kth
Smallest in BST

---

### M4 — Re-index by the thing you're querying · 8 problems

| Trigger | Do this |
|---|---|
| "count subarrays with an exact sum", negatives allowed | Prefix sum → count map. Never a window |
| Subarray with a **divisibility** property | Prefix sums mod k; store the earliest index |
| "for each i, count elements at j>i satisfying an inequality" | Sweep one direction, frequency structure over the **value** axis |
| "top k by a derived key" with the key bounded by n | Bucket by the key; no sort, no heap |
| Pairwise comparison of entities is too expensive | Key on a shared **derived form**; neighbours become a hash lookup |
| Pop by a derived priority with a recency tie-break | Bucket per priority level |
| An O(n) `find` inside a recursion | Precompute value → position map |

Members: Subarray Sum Equals K · Path Sum III · Continuous Subarray Sum · Count of Smaller After
Self · Top K Frequent · Longest Consecutive Sequence · Word Ladder · Construct Tree from Pre+In

**Fails when:** the key isn't stable — if it changes as the scan proceeds and invalidates old
entries, you need a structure with eviction (M6).

---

### M8 — Sort by the factor you want to freeze · 7 problems

| Trigger | Do this |
|---|---|
| Objective = (max or min over the set) × (sum over the set) | Sort by the extremal factor; heap the additive one |
| 2D dominance / chain problem | Sort one axis away → 1D. **The tie-break is the correctness** |
| Weighted interval selection with an O(n²) predecessor scan | Sort by **end**; predecessors become a prefix → binary search |
| "minimum points to stab all intervals" | Sort by end, running intersection |
| Arrange so no two adjacent are equal / spacing constraint | Order by the bottleneck — the most frequent item |

Members: Min Cost to Hire K Workers · Max Performance of a Team · Max Profit in Job Scheduling ·
Russian Doll Envelopes · Min Arrows to Burst Balloons · Reorganize String · Task Scheduler

**Fails when:** you sort by the obvious key. In this corpus the failure is *always the wrong sort
key*, never the absence of sorting — job scheduling by start instead of end, envelopes without the
descending tie-break.

---

### M9 — Rewrite the objective into an equivalent one · 7 problems

| Trigger | Do this |
|---|---|
| "circular" bolted onto an array problem you can solve linearly | Split wrap / non-wrap; wrap = `total − minLinear` |
| A counting problem with an **exact** constraint | `exactly(k) = atMost(k) − atMost(k−1)` |
| A symmetric-partition objective | Fix one side's value from the invariant (total/2) |
| Two players, fixed total | `A − B = 2A − T` → maximise the lead |
| Objects moving/colliding over time | **Never simulate.** What single scalar per element captures it? |
| One objective term is monotone regardless of the choice | It isn't a decision variable; optimise the other greedily |
| A representation whose cost tracks **shape** rather than count | Re-encode |

Members: Max Sum Circular Subarray · Subarrays with K Different · Partition Equal Subset Sum · Gas
Station · Car Fleet · Max Score of a Good Subarray · Serialize/Deserialize Binary Tree

**Fails when:** you can't verify the rewrite. Max Sum Circular's complement formula is wrong for
all-negative input and that degeneracy was missed under an explicit prompt. **Every rewrite needs
one adversarial instance before you build on it.**

---

### M11 — One pass per direction, or carry both extremes · 4 problems

| Trigger | Do this |
|---|---|
| Each element constrained by **both** neighbours / both sides | One pass per direction, combine with max/min |
| An operation that flips ordering (negatives, sign changes) | Carry max **and** min |
| The best answer at a node has a shape the parent can't extend | Return value ≠ recorded value |
| "for every i, an aggregate over all j≠i", division banned | Prefix sweep × suffix sweep |

Members: Candy · Maximum Product Subarray · Product of Array Except Self · Binary Tree Max Path Sum

Small but clean: 4 problems, 4 strong rounds, all self-derived.

---

### M14 — Compose structures to hit per-operation targets · 3 problems

| Trigger | Do this |
|---|---|
| The statement is an **API with per-operation complexity targets**, not a question with an answer | That is the tell |
| One operation needs ordering, another needs lookup, both O(1) | No single structure does both — compose, and store the handle in the map |
| A derived priority that only moves by ±1 | Bucket ladder + a single counter |

Members: LRU Cache · Find Median from Data Stream · Maximum Frequency Stack

---

### D — direction / order modifier · applies to ~35 problems, always second

| Trigger | Do this |
|---|---|
| The value at a position depends on what happens **after** it | Run the DP backwards |
| Your query is over the **suffix** | Scan right to left (prefix → left to right) |
| Dependencies point every way; no index order works | Recurse + memo; there is no static fill order |
| Many sources, few sinks | Run the search **backwards from the sinks** |
| The ordering decision feels complicated | Defer it — process on exit (post-order), then reverse |
| Intervals where you occupy a *point* inside the range | Sweep the **coordinate**, not the items |

**Direction rule: scan away from the side you're querying.** Per the 132 Pattern debrief this is
*"a lookup, not a derivation"* — you have never gotten it right first try, so treat it as memorised.

---

## 2. Families — clusters that share a derivation

Each family cuts across at least three LeetCode tags. The strongest evidence that tags are the
wrong retrieval key: **F4's two members are tagged identically and you still failed to transfer
between them**, while F6 spans Binary Search, Matrix, Tree and Heap and is one algorithm.

| # | Skeleton | Members | What varies |
|---|---|---|---|
| **F1** Regret heap | Process in forced order → take everything, heap the cost → constraint breaks, pop the worst | Min Refueling Stops · Course Schedule III · Furthest Building · Min Intervals to Cover · Max Events | What "worst" means and what popping buys |
| **F2** Mono stack + affordability guard | Build L→R optimistically → pop everything now known worse → guard each pop with a feasibility question | Remove Duplicate Letters · Asteroid Collision · Max Width Ramp · Basic Calculator II · *(unsolved: Remove K Digits, Create Maximum Number)* | The pop guard |
| **F3** Fix the extreme, own the range | Value depends on one element of each subrange → enumerate that element → prev-smaller/next-smaller bounds | Largest Rectangle · Sum of Subarray Minimums · 132 Pattern · Max Width Ramp · Odd Even Jump · Longest Valid Parens | What it's extreme *with respect to*; whether the answer is the popped or retained value |
| **F4** Fix the LAST operation | Removing one element changes adjacency → splitting on the FIRST leaves halves coupled → `dp[l][r] = best over k of cost(l-1,r+1) + dp[l][k-1] + dp[k+1][r]` | Burst Balloons · Min Cost to Cut a Stick · *(adjacent: Remove Boxes, Strange Printer, Matrix Chain)* | The cost term, and max vs min. Nothing else |
| **F5** Sort to freeze, heap the rest | Objective couples a max/min over the set with a sum → sort by the extremal factor → heap the best-k of the additive one | Min Cost to Hire K Workers · Max Performance of a Team · Max Profit Job Scheduling · Russian Doll Envelopes | Whether factor two needs a heap, a binary search, or an LIS |
| **F6** Test a value | Building the answer is expensive, `ok(v)` is a linear scan → verify monotone → `lo<hi, hi=mid, ret lo` | Split Array Largest Sum · Kth Smallest in Matrix · Min Days Bouquets · Kth Smallest Pair Distance · Median of Two Sorted · Count Complete Tree Nodes | The feasibility oracle |
| **F7** Prefix aggregate as hash key | A range quantity is a difference of two prefixes → store prefixes as you scan → look up the partner | Subarray Sum = K · Path Sum III · Continuous Subarray Sum · Count of Range Sum · Num Submatrices = Target | The key (sum / sum mod k / sum in a window) and whether you need equality or a range |
| **F8** Manufacture monotonicity | You want a window but adding elements can re-validate it → find a bounded monotone side-quantity → fix it and enumerate its small range | Longest Substring ≥ K Repeating · Subarrays with K Different · Shortest Subarray Sum ≥ K | Enumerate the cap, subtract two at-mosts, or abandon the window for prefix+deque |
| **F9** Solve once, then delta | Answer wanted at every position, one costs O(n) → compute once plus what the delta needs → O(1) move to a neighbour | Sum of Distances in Tree · Making a Large Island · Number of Islands II · Partition Labels | Whether the delta is arithmetic, a lookup, or an incremental merge |
| **F10** Add the dimension the future cares about | Two arrivals at a place aren't interchangeable → name what differs → it joins the visited/memo key | Shortest Path w/ Obstacles · Cheapest Flights K Stops · Min Cost in Time · Best Time IV · Word Search *(the negative case)* | What the extra dimension is, and whether it makes memoisation possible or impossible |
| **F11** Two passes, one per direction | Both sides constrain it, or the operation is non-monotone → satisfy one side forward → satisfy the other backward, combining | Candy · Product of Array Except Self · Maximum Product Subarray | Whether the second pass folds into the output array or into a scalar pair |

---

## 3. Your failure profile

**Which moves beat you** — 29 derivation-failure rounds across 26 distinct problems (132 Pattern
contributes three rounds):

M6 6/9 (**67%**) · M12 2/4 (50%) · M2 5/11 (45%) · M4 3/8 (38%) · M8 2/7 (29%) · M9 2/7 (29%) ·
M10 2/9 (22%) · M1 3/14 (21%) · M7 3/18 (17%) · M5 1/12 (8%) · M11+M14 0/7 (0%)

**M6 has been your worst move for the entire recorded history**, failing identically four times
across three months: Min Refueling Stops 05/06 (*"didn't see the 'defer the decision' insight on
his own"*) → Course Schedule III 05/11 (*"the greedy + heap swap insight required hints"*) →
Furthest Building 05/17 (*"started with a flawed local greedy; two counterexamples to reach
defer-ladder + heap"*) → Remove Duplicate Letters 07/28 (*"you diagnosed the insufficiency as a
coverage problem rather than an expressiveness problem"*). Nothing else in the corpus repeats this
cleanly.

**M2 fails differently — you recognise the trigger and discard it.** Min Days to Make M Bouquets:
*"after correctly framing 'minimize the maximum,' said 'there is no way I can think of to minimise
the maximum, so this characterisation may be wrong.'"*

Underneath the move-level ranking, four mechanisms:

**(1) You get stuck when the next step requires abandoning a frame, not refining one.** Your best
moves — M5 (8%), M11 and M14 (0%) — all *refine* a formulation you already have. Your worst — M6,
M2, M12 — all require **throwing out a commitment**: my greedy was wrong and needs an undo; stop
constructing the answer and start testing it; stop running the algorithm n times. The 132 Pattern
debrief names the mechanism: *"a plausible-sounding wrong answer is the dangerous kind — it
terminated his search."* This is why you go silent. You are not out of ideas; you are out of ideas
*inside the frame you committed to*, and you have no procedure for leaving it.

**(2) You stop at the diagnosis instead of converting it into the next question.** Longest
Substring ≥ K Repeating: *"at minute 17 you correctly said 'there is no condition for shrinking.'
That is exactly the moment to ask 'what could I add that WOULD give me one?' Diagnosing a dead end
and then not turning it into the next question is where this round was lost."* Same shape in
Longest Repeating Char Replacement, Odd Even Jump, Longest Valid Parentheses. You produce correct
diagnoses and treat them as terminal.

**(3) You reason abstractly when your reliable skill is reading a concrete instance.** 132 Pattern:
*"the trace wasn't a verification step, it was the derivation step, and he skipped it."* RegEx
Matching: *"a concrete failing example was handed to him and he would not run it. Both times the
trace would have produced the insight, not merely confirmed it."* **You have never once traced a
small case and failed to extract the structure.** You just don't do it unprompted.

**(4) Your failures are retrieval failures, not knowledge gaps.** Burst Balloons (07/28) is the
identical recurrence to Min Cost to Cut a Stick, solved cleanly 07/08 — twenty days, not
recognised. Longest Substring ≥ K Repeating (07/28) is the same bounded-cap move as Subarrays with
K Different, solved 06/12. *"The fix is deliberate retrieval practice, not more new problems."*

**One sentence:** you commit to a frame early, treat a plausible-sounding wrong answer as settled,
correctly diagnose the dead end, and then stop — when the two things that reliably unstick you
(writing down a bad candidate state, hand-tracing a 4-element case) are available at any moment and
almost never done unprompted.

---

## 4. The process

### Minutes 0–2 · Clarify and mine

Ask these four out loud, every round. You have skipped this in 28 sessions.

1. What are the degenerate sizes, and what do they return?
2. Duplicates possible? Comparisons strict or not?
3. **Can I mutate the input?**
4. What is `n`, and what complexity does that imply?

Then say one sentence per constraint: **"`X` means I should look for `Y`."** Use the M13 table. Do
not skip because it feels obvious — this is where four of your worst rounds were lost.

### Minutes 2–5 · Brute force, then name the waste

1. State the brute force and its complexity. Out loud. Always.
2. State the target complexity implied by `n`.
3. Say **one sentence**: *"For every ___ it re-scans ___ that it already knew."*

That sentence selects your move. Nothing else in this document matters if you skip it.

- re-scans a range aggregate over a shifting set → **M5**
- re-scans subranges whose value depends on one element → **M1**
- re-evaluates the same (position, extra-thing) pair → **M7**
- produces every candidate when you only need to test one → **M2**
- runs the whole algorithm once per position → **M12**
- compares every pair of entities → **M4**
- re-decides a choice you can't yet evaluate → **M6**
- re-simulates objects moving over time → **M9**

### Minutes 5–12 · Commit, write the state, sanity-check

1. Name the move out loud: *"I'm going to fix the peak index and query each side."*
2. **Write the state/invariant as one sentence, quantified over the whole scan.** If it needs the
   word "currently" or "this step," it's a loop-body statement, not an invariant — rewrite it.
3. Run the two free checks:
   - **Window?** *"If [l,r] fails, must every window containing it fail?"* No → M2 or a deque.
   - **DP?** *For each branch: what did I consume from each input? Subtract exactly that.* Then: do
     two semantically different branches have identical formulas? If yes, one is wrong.
4. Pick the direction (D). **Scan away from the side you query.**
5. Trace 3–4 elements by hand before writing code.

### Minute 8 · You are stuck. Do exactly this.

**No hint request. No silence. Say these four lines out loud, however bad:**

```
1. A subproblem is:  solve(...) = ...
2. The state is (...) because I need ... to evaluate it.
3. It fails / I'm unsure because ...
4. The smallest case where I can't decide is ...
```

Line 3 is the only line an interviewer can help with, and it is the line you never produce.
**A wrong subproblem written down is worth more than a right one withheld.**

Then, in order — stop as soon as one works:

1. **Shrink to n = 3 or 4 and compute the answer by hand, on the board.** Every structural insight
   in this corpus was visible in the smallest non-trivial case.
2. **Re-read the constraints and name one you have not spent.** There is always one.
3. **Say your diagnosis, then immediately turn it into a question.** Not *"there's no shrink
   condition"* → *"what could I add that WOULD give me one?"* Not *"I can't minimise the maximum"*
   → *"can I check one candidate maximum?"* Not *"I'm not generating enough candidates"* → *"is my
   move set missing an operation?"*
4. **The frame question:** *"which index did I fix, and what is the residual query?"* Write it as a
   literal sentence. Still 2-D → wrong index.
5. **The abandonment question:** *"what am I assuming that the problem never said?"* Usually: that
   the decision is irreversible (→ M6), that the process runs forward (→ D), that you must
   construct the answer (→ M2), or that the input is a general graph (→ it's a tree).

If none of those moves you in 90 seconds, say: *"I'm stuck. Here's my brute force, here's the waste
I've identified, here's the frame I tried and why it fails."* That sentence scores. Silence and
"give me a hint" do not.

### Minute 12+ · Before you say "done"

1. Run the **hostile** input, not the friendly one: empty / n=1 / all-equal / negatives / ties /
   answer-at-an-extreme. Four consecutive rounds shipped bugs masked by a friendly self-chosen test.
2. Trace **the actual code**, reading your own loop bounds — not your intent.
3. Name every array you allocated and where each complexity term comes from.
4. Check the container: `std::map` is never O(1); `operator[]` is never a pure read.
5. Sanity-check magnitudes against the constraints for overflow.

**Two standing rules from the transcripts:** an input handed to you by the interviewer is an
instruction, not a suggestion — run it before typing anything else. And when asked to trace, trace;
never substitute a complexity claim for a trace.

---

## 5. Pre-round card

```
================================================================
0-2  ASK: degenerate sizes? duplicates/strictness? CAN I MUTATE?
     n = ? -> target complexity = ?
     One line per constraint: "X means look for Y."

2-5  Brute force + its O(). Then ONE sentence:
     "For every ___ it re-scans ___ it already knew."

     range aggregate, shifting set ...... MAINTAIN IT (deque/heap/prefix)
     subranges, value from 1 element .... FIX THAT ELEMENT (mono stack)
     same (place, extra-thing) pair ..... RESIZE THE STATE
     builds all candidates .............. TEST A VALUE (lo<hi, hi=mid, ret lo)
     runs whole algo per position ....... SOLVE ONCE + DELTA
     compares every pair ................ RE-INDEX BY THE KEY
     re-decides an unevaluable choice ... COMMIT + RETRACT (regret heap)
     simulates motion ................... ONE SCALAR PER ELEMENT
     transitive relations / spreading ... GRAPH -> name the traversal
     max/min x sum objective ............ SORT BY THE EXTREMAL FACTOR
     both neighbours constrain it ....... ONE PASS PER DIRECTION
     API with per-op targets ............ COMPOSE TWO STRUCTURES

5-12 Say the move. Write the invariant (whole scan, not "currently").
     Window?  "If [l,r] fails, must every superset fail?"  no -> enumerate a cap
     DP?      "What did each branch consume? Subtract that."
              Two branches, same formula = bug.
     Direction: SCAN AWAY FROM THE SIDE YOU QUERY.
     Trace 3 elements before coding.

  8  STUCK - no hints, no silence. Say these four:
       1. A subproblem is: solve(...) = ...
       2. The state is (...) because I need ... to evaluate it
       3. It fails / I'm unsure because ...
       4. Smallest case I can't decide: ...
     Then, in order:
       a. n=3 by hand ON THE BOARD          <- this always works for you
       b. name a constraint you haven't spent
       c. diagnosis -> question ("what WOULD give me one?")
       d. which index did I fix? residual query still 2-D? wrong index
       e. what am I assuming? (irreversible / forward / must construct / general graph)

 12+ HOSTILE input: empty, n=1, all-equal, negatives, ties, extreme
     Trace THE CODE, not your intent. Name every array. map != O(1).
================================================================
YOUR THREE WORST MOVES - check for them explicitly:
  M6  regret heap   67% fail   "learn too late whether a choice was right"
  M12 solve+delta   50% fail   "the answer, for EVERY position"
  M2  test a value  45% fail   "min the max / kth / smallest v such that"

YOUR TELL: you diagnose the dead end correctly, then stop.
           The diagnosis is the setup for the next question, not the end.
TREE != general graph.  "at most k" => greedy unsafe.  negatives => no window.
n<=300 => cubic => interval DP on the LAST operation.
================================================================
```

---

## 6. The three drills the evidence supports

1. **F4 retrieval.** Re-derive Min Cost to Cut a Stick and Burst Balloons back to back, cold. You
   solved one and failed the other twenty days apart. The fix for a retrieval failure is deliberate
   re-derivation of solved problems, not new problems.
2. **M6, deliberately.** Re-derive Min Refueling Stops, Course Schedule III, Furthest Building and
   Remove Duplicate Letters in one sitting, writing only the pop/regret guard for each. Your 67%
   move, failed identically four times across three months.
3. **The minute-8 script, out loud, under a timer.** Take any unsolved Hard, set a timer for 8
   minutes, and at the bell say the four lines regardless of state. The corpus says your derivation
   works whenever you instantiate it; the failure is that you don't.
