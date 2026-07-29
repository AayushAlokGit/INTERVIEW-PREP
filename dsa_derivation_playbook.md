# DSA Derivation Playbook

Induced from 107 solved-problem rounds (2026-04-29 → 2026-07-28) in `transcripts/<Y>/<M>/<D>/dsa/`.
Organised by *what you notice* → *what you do*, not by LeetCode tag.

**Read order under time pressure:** §4 card → §3 process → §1 moves.

Failure rates below are **derivation failure** (the pivotal move was hinted, handed over, or never
arrived): 29 rounds over 26 problems. Overall round rating is the wrong lens — Complexity and
Communication scores stay high even when the algorithm never came. Denominators below M2 are small;
the gap between 29% and 21% is noise. M11/M12/M14 rest on 3–4 problems each.

---

## 1. The moves

| Move | n | Fail |
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

Cross-cutting: **D — pick the direction/order** (~35 problems, always a *second* decision).
Meta: **M13 — spend the constraint** (minute 2, selects among the rest). No M3 — it was
"run it in the order that makes the unknown already known," which collapsed into D.
Unclassified: String to Integer (atoi) — the brute force *is* optimal; the difficulty is the spec.

Ordered below by how badly they beat you.

---

### M13 — Spend the constraint · meta-move, minute 2

Produces no algorithm; tells you **which one to look for**. The most frequently named miss in the corpus.

| You were told | Do this |
|---|---|
| A target complexity tighter than your brute force | Ask what the bound **forbids**. `O(n)` kills sorting and heaps |
| A tiny bounded universe (26, 32, 10, 9, 100) | Loop around it, or index an array by it |
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

Missed in: Longest Consecutive Sequence (*"the O(n) requirement IS the hint"*), Burst Balloons
(`n ≤ 300` specifies cubic), Longest Substring ≥ K Repeating (26 read past for 23 min), Odd Even
Jump, Min Cost to Cut a Stick (`n ≤ 10^6` vs an `n×n` allocation).

---

### M6 — Commit optimistically, then retract · 9 problems · ⚠ 67%, your worst

| Trigger | Do this |
|---|---|
| You only learn whether a choice was right **after** you've passed it | Take everything, keep an undo structure |
| Two resource types, one strictly better but scarcer | Spend the scarce one greedily, demote the worst use later |
| "Maximise the count under a cumulative budget", forced arrival order | Take-then-regret max-heap |
| Building a sequence left-to-right under a lexicographic objective | Monotonic stack, pop while worse |
| A greedy that *almost* works but breaks on a counterexample | Don't abandon greed — add an **undo** |
| Elements interact only with their nearest surviving neighbour | Stack of survivors |

**Diagnostic when a greedy fails:** *is my candidate set too small, or is my move set too small?*
Coverage vs expressiveness failure have opposite fixes. Move set → add `pop`.
**The pop guard is always a feasibility question**, problem-specific ("does this letter occur
later?", "do I have budget left?"). Getting the guard right *is* the correctness argument.

Members: Min Refueling Stops · Course Schedule III · Furthest Building · Min Intervals to Cover
Target · Max Events · Maximum Width Ramp · Asteroid Collision · Remove Duplicate Letters · Basic
Calculator II

**Fails when:** retraction isn't legal — an earlier commitment already changed the world
irreversibly. Then you need real DP (M7). *"At most k transactions"* + non-overlap (Best Time IV)
is the canonical trap.

---

### M2 — Search the answer's value instead of constructing it · 11 problems · ⚠ 45%

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
count(d) ≥ k."* **Anchor rule:** compare against a fixed end (`nums[hi]`), never a moving one.

Members: Split Array Largest Sum · Kth Smallest in Sorted Matrix · Median of Two Sorted Arrays ·
Min Days to Make M Bouquets · Kth Smallest Pair Distance · Find K Closest · Count Complete Tree
Nodes · Search in Rotated Sorted Array · First & Last Position · Find Min in Rotated II · Longest
Substring with At Least K Repeating

**Fails when:** `ok()` isn't monotone, or the predicate is uninformative on ties (Find Min in
Rotated II — handle separately, expect to lose the log bound). Your specific failure mode is
recognising the trigger and *discarding* it: Min Days to Make M Bouquets — *"after correctly
framing 'minimize the maximum,' said 'there is no way I can think of to minimise the maximum, so
this characterisation may be wrong.'"*

---

### M12 — Compute one answer, reach the neighbour by a delta · 4 problems · ⚠ 50% (thin)

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
**Branch-identity check:** two semantically different branches with identical formulas = a bug, every time.

Members (add): Validate BST · Shortest Path w/ Obstacles · Cheapest Flights K Stops · Min Cost in
Time · Best Time IV · Restore IP · Word Search · RegEx Matching · Dungeon Game · Longest Increasing Path
(drop/collapse): Decode Ways · Cherry Pickup II · Stone Game III · Word Break · Edit Distance ·
Coin Change · Palindrome Partitioning · Combination Sum

**Fails when:** the state is genuinely path-dependent (Word Search) — no memo exists. Also when a
param's numeric range is huge but its real domain is small (Cut a Stick sized the memo `n×n` = 10^12
when the domain was the cut list).

---

### M1 — Fix the element that decouples the rest · 14 problems · 21%

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

Members: Largest Rectangle · Sum of Subarray Minimums · Maximal Square · Longest Valid Parentheses ·
Longest Palindromic Substring · Count of Range Sum · Min Cost to Cut a Stick · Burst Balloons ·
Number of Submatrices Sum to Target · 132 Pattern ×3 · Insert Interval · Next Permutation

**Fails when:** the objective isn't determined by a single element (sum-based subarray → M4 or M5).

---

### M5 — Carry the inner loop's answer forward · 12 problems · your strongest, 8%

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

**Precondition, before writing any window:** *"if [l,r] fails, must every window containing it
fail?"* If no → M2 (enumerate a bounded cap) or prefix sums + deque.

Members: LS At Most K Distinct · Min Window Substring · Longest Repeating Char Replacement ·
Subarray Product < K · Sliding Window Maximum · Jump Game VI · Shortest Subarray Sum ≥ K · Jump
Game II · Kth Largest · Smallest Range K Lists · LIS · Odd Even Jump

**Fails when:** the predicate isn't monotone (Longest Substring ≥ K Repeating kills the window
outright), or negatives are allowed (→ prefix + deque, or M4).

---

### M10 — Turn the relation into a graph, name the traversal · 9 problems · 22%

| Trigger | Do this |
|---|---|
| Pairwise relations that **compose transitively** (ratios, rates, offsets) | Weighted graph, path aggregate |
| Recover a total order from local pairwise evidence | DAG + topological sort |
| Something **spreading** from many origins at once | Multi-source BFS, levels = time |
| Shortest transformation / fewest steps, unweighted | BFS |
| "min over paths of the max on the path" | Dijkstra with a min-heap frontier |
| Connect everything at minimum total cost, no ordering required | MST (dense → array Prim) |
| Use every edge exactly once | Eulerian path (Hierholzer: post-order + reverse) |
| Online connectivity updates | Union-Find |
| Values are valid indices into their own array | Functional graph → cycle detection |

**Anti-trigger:** the input is a **tree**. A tree is almost never a general-graph problem — Sum of
Distances in Tree opened with Floyd–Warshall on an unweighted tree.

Members: Evaluate Division · Alien Dictionary · Rotting Oranges · Trapping Rain Water II · Min Cost
to Connect All Points · Find the Duplicate Number · Pacific Atlantic · Reconstruct Itinerary · Kth
Smallest in BST

---

### M4 — Re-index by the thing you're querying · 8 problems · 38%

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

### M8 — Sort by the factor you want to freeze · 7 problems · 29%

| Trigger | Do this |
|---|---|
| Objective = (max or min over the set) × (sum over the set) | Sort by the extremal factor; heap the additive one |
| 2D dominance / chain problem | Sort one axis away → 1D. **The tie-break is the correctness** |
| Weighted interval selection with an O(n²) predecessor scan | Sort by **end**; predecessors become a prefix → binary search |
| "minimum points to stab all intervals" | Sort by end, running intersection |
| Arrange so no two adjacent are equal / spacing constraint | Order by the bottleneck — the most frequent item |

Members: Min Cost to Hire K Workers · Max Performance of a Team · Max Profit in Job Scheduling ·
Russian Doll Envelopes · Min Arrows to Burst Balloons · Reorganize String · Task Scheduler

**Fails when:** you sort by the obvious key. The failure here is *always the wrong sort key*, never
the absence of sorting — job scheduling by start instead of end, envelopes without the descending
tie-break.

---

### M9 — Rewrite the objective into an equivalent one · 7 problems · 29%

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
all-negative input. **Every rewrite needs one adversarial instance before you build on it.**

---

### M11 — One pass per direction, or carry both extremes · 4 problems · 0%

| Trigger | Do this |
|---|---|
| Each element constrained by **both** neighbours / both sides | One pass per direction, combine with max/min |
| An operation that flips ordering (negatives, sign changes) | Carry max **and** min |
| The best answer at a node has a shape the parent can't extend | Return value ≠ recorded value |
| "for every i, an aggregate over all j≠i", division banned | Prefix sweep × suffix sweep |

Members: Candy · Maximum Product Subarray · Product of Array Except Self · Binary Tree Max Path Sum

---

### M14 — Compose structures to hit per-operation targets · 3 problems · 0%

| Trigger | Do this |
|---|---|
| The statement is an **API with per-operation complexity targets**, not a question with an answer | That is the tell |
| One operation needs ordering, another needs lookup, both O(1) | No single structure does both — compose, store the handle in the map |
| A derived priority that only moves by ±1 | Bucket ladder + a single counter |

Members: LRU Cache · Find Median from Data Stream · Maximum Frequency Stack

---

### D — direction / order modifier · ~35 problems, always second

| Trigger | Do this |
|---|---|
| The value at a position depends on what happens **after** it | Run the DP backwards |
| Your query is over the **suffix** | Scan right to left (prefix → left to right) |
| Dependencies point every way; no index order works | Recurse + memo; there is no static fill order |
| Many sources, few sinks | Run the search **backwards from the sinks** |
| The ordering decision feels complicated | Defer it — process on exit (post-order), then reverse |
| Intervals where you occupy a *point* inside the range | Sweep the **coordinate**, not the items |

**Direction rule: scan away from the side you're querying.** *"A lookup, not a derivation"* — you
have never gotten it right first try, so treat it as memorised.

---

## 2. Families and failure profile

Families cut across LeetCode tags — F4's two members are tagged identically and you still failed to
transfer between them; F6 spans Binary Search, Matrix, Tree and Heap and is one algorithm.

| # | Skeleton | Members | What varies |
|---|---|---|---|
| **F1** Regret heap | Forced order → take everything, heap the cost → constraint breaks, pop the worst | Min Refueling Stops · Course Schedule III · Furthest Building · Min Intervals to Cover · Max Events | What "worst" means |
| **F2** Mono stack + affordability guard | Build L→R optimistically → pop what's now known worse → guard each pop with a feasibility question | Remove Duplicate Letters · Asteroid Collision · Max Width Ramp · Basic Calculator II · *(unsolved: Remove K Digits, Create Maximum Number)* | The pop guard |
| **F3** Fix the extreme, own the range | Value depends on one element per subrange → enumerate it → prev/next-smaller bounds | Largest Rectangle · Sum of Subarray Minimums · 132 Pattern · Max Width Ramp · Odd Even Jump · Longest Valid Parens | Extreme *w.r.t. what*; popped vs retained value |
| **F4** Fix the LAST operation | Removal changes adjacency → splitting on the FIRST leaves halves coupled → `dp[l][r] = best over k of cost(l-1,r+1) + dp[l][k-1] + dp[k+1][r]` | Burst Balloons · Min Cost to Cut a Stick · *(adjacent: Remove Boxes, Strange Printer, Matrix Chain)* | The cost term, max vs min |
| **F5** Sort to freeze, heap the rest | Max/min over the set × a sum → sort by the extremal factor → heap best-k of the additive one | Min Cost to Hire K Workers · Max Performance of a Team · Max Profit Job Scheduling · Russian Doll Envelopes | Heap vs binary search vs LIS |
| **F6** Test a value | Building is expensive, `ok(v)` is a linear scan → verify monotone → `lo<hi, hi=mid, ret lo` | Split Array Largest Sum · Kth Smallest in Matrix · Min Days Bouquets · Kth Smallest Pair Distance · Median of Two Sorted · Count Complete Tree Nodes | The feasibility oracle |
| **F7** Prefix aggregate as hash key | Range quantity = difference of two prefixes → store prefixes as you scan → look up the partner | Subarray Sum = K · Path Sum III · Continuous Subarray Sum · Count of Range Sum · Num Submatrices = Target | The key; equality vs range |
| **F8** Manufacture monotonicity | Adding elements can re-validate the window → find a bounded monotone side-quantity → fix it, enumerate its small range | Longest Substring ≥ K Repeating · Subarrays with K Different · Shortest Subarray Sum ≥ K | Enumerate the cap, subtract two at-mosts, or drop the window for prefix+deque |
| **F9** Solve once, then delta | Answer at every position, one costs O(n) → compute once + what the delta needs | Sum of Distances in Tree · Making a Large Island · Number of Islands II · Partition Labels | Arithmetic delta, lookup, or incremental merge |
| **F10** Add the dimension the future cares about | Two arrivals aren't interchangeable → name what differs → it joins the visited/memo key | Shortest Path w/ Obstacles · Cheapest Flights K Stops · Min Cost in Time · Best Time IV · Word Search *(negative case)* | Whether it makes memoisation possible or impossible |
| **F11** Two passes, one per direction | Both sides constrain it, or the op is non-monotone → satisfy one side forward, the other backward | Candy · Product of Array Except Self · Maximum Product Subarray | Output array vs scalar pair |

**Four mechanisms behind the move-level rates:**

1. **You get stuck when the next step requires abandoning a frame, not refining one.** M5/M11/M14
   (0–8%) all *refine* a formulation you have. M6/M2/M12 all require **throwing out a commitment**.
   *"A plausible-sounding wrong answer is the dangerous kind — it terminated his search."* You are
   not out of ideas; you are out of ideas *inside the frame you committed to*.
2. **You stop at the diagnosis instead of converting it into the next question.** Longest Substring
   ≥ K Repeating: *"at minute 17 you correctly said 'there is no condition for shrinking.' That is
   exactly the moment to ask 'what could I add that WOULD give me one?'"* Same in Longest Repeating
   Char Replacement, Odd Even Jump, Longest Valid Parentheses.
3. **You reason abstractly when your reliable skill is reading a concrete instance.** *"The trace
   wasn't a verification step, it was the derivation step, and he skipped it."* **You have never
   once traced a small case and failed to extract the structure** — you just don't do it unprompted.
4. **Your failures are retrieval failures, not knowledge gaps.** Burst Balloons (07/28) is the
   identical recurrence to Min Cost to Cut a Stick, solved cleanly 07/08. Longest Substring ≥ K
   Repeating (07/28) is the same bounded-cap move as Subarrays with K Different (06/12).

**M6 has been your worst move for the entire recorded history**, failing identically four times
across three months: Min Refueling Stops 05/06 → Course Schedule III 05/11 → Furthest Building
05/17 → Remove Duplicate Letters 07/28 (*"you diagnosed the insufficiency as a coverage problem
rather than an expressiveness problem"*).

---

## 3. The process

### Minutes 0–2 · Clarify and mine

Ask out loud, every round (skipped in 28 sessions):

1. Degenerate sizes, and what do they return?
2. Duplicates possible? Comparisons strict or not?
3. **Can I mutate the input?**
4. What is `n`, and what complexity does that imply?

Then one sentence per constraint: **"`X` means I should look for `Y`."** Use the M13 table.

### Minutes 2–5 · Brute force, then name the waste

State the brute force + its complexity, out loud. State the target complexity implied by `n`. Then
say **one sentence**: *"For every ___ it re-scans ___ that it already knew."* That sentence selects
your move; nothing else here matters if you skip it.

- range aggregate over a shifting set → **M5** · subranges valued by one element → **M1**
- same (position, extra-thing) pair → **M7** · builds every candidate to test one → **M2**
- whole algorithm once per position → **M12** · every pair of entities → **M4**
- re-decides a choice you can't yet evaluate → **M6** · re-simulates motion → **M9**

### Minutes 5–12 · Commit, write the state, sanity-check

1. Name the move out loud: *"I'm going to fix the peak index and query each side."*
2. **Write the state/invariant as one sentence over the whole scan.** If it needs "currently" or
   "this step," it's a loop-body statement — rewrite it.
3. Two free checks — **Window?** *"If [l,r] fails, must every window containing it fail?"* No → M2
   or a deque. **DP?** *Per branch: what did I consume from each input? Subtract exactly that.* Then:
   two semantically different branches with identical formulas → one is wrong.
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

1. **Shrink to n = 3 or 4 and compute by hand, on the board.** Every structural insight in this
   corpus was visible in the smallest non-trivial case.
2. **Name one constraint you have not spent.** There is always one.
3. **Turn the diagnosis into a question.** Not *"there's no shrink condition"* → *"what could I add
   that WOULD give me one?"* Not *"I can't minimise the maximum"* → *"can I check one candidate
   maximum?"* Not *"I'm not generating enough candidates"* → *"is my move set missing an operation?"*
4. **The frame question:** *"which index did I fix, and what is the residual query?"* Still 2-D →
   wrong index.
5. **The abandonment question:** *"what am I assuming that the problem never said?"* Usually: the
   decision is irreversible (→ M6), the process runs forward (→ D), you must construct the answer
   (→ M2), or the input is a general graph (→ it's a tree).

If none of those moves you in 90 seconds: *"I'm stuck. Here's my brute force, here's the waste I've
identified, here's the frame I tried and why it fails."* That sentence scores. Silence doesn't.

### Minute 12+ · Before you say "done"

1. Run the **hostile** input: empty / n=1 / all-equal / negatives / ties / answer-at-an-extreme.
   Four consecutive rounds shipped bugs masked by a friendly self-chosen test.
2. Trace **the actual code**, reading your own loop bounds — not your intent.
3. Name every array you allocated and where each complexity term comes from.
4. Check the container: `std::map` is never O(1); `operator[]` is never a pure read.
5. Sanity-check magnitudes for overflow.

**Two standing rules:** an input handed to you by the interviewer is an instruction — run it before
typing anything else. And when asked to trace, trace; never substitute a complexity claim for a trace.

---

## 4. Pre-round card

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

## 5. The three drills

1. **F4 retrieval.** Re-derive Min Cost to Cut a Stick and Burst Balloons back to back, cold. You
   solved one and failed the other twenty days apart. The fix for a retrieval failure is deliberate
   re-derivation of solved problems, not new problems.
2. **M6, deliberately.** Re-derive Min Refueling Stops, Course Schedule III, Furthest Building and
   Remove Duplicate Letters in one sitting, writing only the pop/regret guard for each.
3. **The minute-8 script, out loud, under a timer.** Any unsolved Hard, timer at 8 minutes, say the
   four lines regardless of state. Your derivation works whenever you instantiate it; the failure is
   that you don't.
