# DSA Pattern Speedrun

Cram sheet: the ~16 patterns that cover the large majority of interview problems, each shown as a
**ladder** — brute force → each optimisation step, with *the observation that unlocks the step*.
The observation is the thing you say out loud in an interview. The code is secondary.

Companion to `../dsa_derivation_playbook.md` (organised by *derivation move*). This file is
organised by *pattern*, for a fast pre-round refresh.

---

## 0. The 90-second triage

Read the constraints, then the shape of the answer.

### Constraint → intended complexity

| n | Budget | What that allows |
|---|---|---|
| ≤ 12 | O(n!) | Permutation backtracking |
| ≤ 20–25 | O(2^n · n) | Subset backtracking, **bitmask DP** |
| ≤ 100 | O(n^4) / O(n^3) | Floyd–Warshall, interval DP |
| ≤ 500 | O(n^3) | Interval DP (Burst Balloons, MCM) |
| ≤ 5,000 | O(n^2) | 2D DP over pairs (edit distance, LCS, LIS n²) |
| ≤ 10^5 | O(n log n) | Sort, heap, binary search, segment tree |
| ≤ 10^6–10^7 | O(n) | Single pass, two pointers, counting, sieve |
| ≤ 10^9 (a *value*, not a size) | O(log n) | **Binary search the answer**, math |

The inverse read is the useful one: *"they said O(n) → sorting and heaps are dead → I need a hash
map, a monotonic structure, or a two-pointer invariant."*

### Phrase → pattern

| You read | Reach for |
|---|---|
| "contiguous subarray/substring", all values **positive** | Sliding window |
| "subarray sum = k" with **negatives** | Prefix sum + hash map |
| "k distinct" / "at most k" over a window | Variable window + counter map |
| Sorted input, or "pair/triplet summing to X" | Two pointers from both ends |
| "minimise the maximum" / "maximise the minimum" | Binary search on the answer |
| "kth smallest/largest", counting `≤ v` is easy | Binary search on value space |
| "top k" / "k closest" / streaming | Heap of size k |
| "next greater / previous smaller / span" | Monotonic stack |
| "max/min of every window of size k" | Monotonic deque |
| Intervals, meetings, ranges | Sort by start or end, then sweep |
| "number of ways" / "min cost to reach" | DP |
| "all combinations / all valid X" | Backtracking |
| Grid + shortest path, unweighted | BFS |
| Grid + connected regions | DFS / flood fill / Union-Find |
| Dependencies, ordering, cycles | Topological sort |
| Weighted shortest path, non-negative | Dijkstra |
| Dynamic connectivity, "are these merged" | Union-Find |
| Prefix matching, word dictionaries | Trie |
| Values in `[1..n]`, find missing/duplicate | Cyclic sort / index-as-hash |
| O(1) extra space demanded on an array | Mutate the input as your hash table |

---

## 1. Two pointers

**Trigger:** sorted array, or an invariant that lets you *prove* moving one side can never lose the
answer.

### Ladder — Two Sum on a sorted array

```
Brute:   for i, for j>i: a[i]+a[j]==t                  O(n^2)
Better:  for i: binary search t-a[i]                   O(n log n)
Optimal: l=0, r=n-1; sum<t -> l++, sum>t -> r--        O(n), O(1) space
```

**Unlock:** if `a[l]+a[r] < t`, then `a[l]` paired with *anything* is still ≤ that sum, so `l` can
never be part of a solution — you discard a whole row of the pair-matrix per step, not one cell.

### Ladder — Container With Most Water

```
Brute:   all pairs, area = (j-i)*min(h[i],h[j])        O(n^2)
Optimal: two ends, always move the SHORTER wall        O(n)
```

**Unlock:** width only shrinks. Moving the taller wall can never increase `min(...)`, so every pair
involving the shorter wall is already maximised at the current width.

### Ladder — 3Sum

```
Brute:   triple loop                                   O(n^3)
Optimal: sort; fix i; two-pointer the suffix           O(n^2)
```

**Unlock:** fixing one element reduces an unsolved problem to a solved one. This is the single most
reusable move in the whole sheet.

### Skeleton

```python
def two_sum_sorted(a, t):
    l, r = 0, len(a) - 1
    while l < r:
        s = a[l] + a[r]
        if s == t: return (l, r)
        if s < t: l += 1
        else:     r -= 1
    return None
```

**Traps:** duplicate skipping in 3Sum (`while l < r and a[l] == a[l+1]: l += 1`) goes *after* you
record a hit, not before. Two pointers on **unsorted** data is only valid with an independent
monotonic argument (Container With Most Water is the example).

---

## 2. Sliding window

**Trigger:** contiguous window + a metric that is **monotone** as the window grows. All-positive
values is the classic guarantee. Negatives kill it → go to prefix sums instead.

### Fixed window

```python
s = sum(a[:k]); best = s
for i in range(k, len(a)):
    s += a[i] - a[i-k]           # add the new, drop the old
    best = max(best, s)
```

**Unlock:** the O(n·k) recompute shares all but two elements between consecutive windows.

### Variable window — longest valid

```
Brute:   all substrings, validate each                 O(n^3)
Better:  incremental validity while extending          O(n^2)
Optimal: expand right always, shrink left while        O(n)
         invalid — each index enters and leaves once
```

```python
def longest_no_repeat(s):
    last, l, best = {}, 0, 0
    for r, ch in enumerate(s):
        if ch in last and last[ch] >= l:
            l = last[ch] + 1          # jump, don't crawl
        last[ch] = r
        best = max(best, r - l + 1)
    return best
```

### Variable window — shortest valid (Minimum Window Substring)

```python
from collections import Counter

def min_window(s, t):
    need, missing = Counter(t), len(t)
    l, best = 0, (float('inf'), 0, 0)
    for r, ch in enumerate(s):
        if need[ch] > 0: missing -= 1
        need[ch] -= 1
        while missing == 0:                        # valid -> try to shrink
            if r - l < best[0]: best = (r - l, l, r)
            need[s[l]] += 1
            if need[s[l]] > 0: missing += 1
            l += 1
    return "" if best[0] == float('inf') else s[best[1]:best[2]+1]
```

**The "exactly k" trick:** `exactly(k) = atMost(k) - atMost(k-1)`. A direct "exactly k" window is
not monotone; "at most k" is. Use it for *Subarrays with K Different Integers* and *Binary Subarrays
With Sum*.

**Traps:** the shrink condition is a `while`, never an `if`. Record `best` at the right moment —
longest → after expanding; shortest → inside the shrink loop.

---

## 3. Prefix sums + hash map

**Trigger:** subarray sums with **negative** numbers, or many range-sum queries.

### Ladder — Subarray Sum Equals K

```
Brute:   all (i,j), sum the inner range                O(n^3)
Better:  running sum in the inner loop                 O(n^2)
Optimal: seen[prefix] counts; res += seen[p - k]       O(n)
```

**Unlock:** `sum(i..j) = P[j] - P[i-1]`, so *"is there a subarray ending at j summing to k"* becomes
*"have I seen the prefix `P[j] - k` before"* — a lookup, not a scan.

```python
from collections import defaultdict

def subarray_sum(a, k):
    seen = defaultdict(int); seen[0] = 1
    p = res = 0
    for x in a:
        p += x
        res += seen[p - k]
        seen[p] += 1
    return res
```

`seen[0] = 1` seeds the empty prefix — without it you miss every subarray starting at index 0.

**Variants worth 30 seconds each:**

- Sum divisible by k → key on `p % k`, normalising negatives with `(p % k + k) % k`.
- *Longest* subarray with sum k → store the **first** index of each prefix, not a count.
- Equal 0s and 1s → map `0 -> -1`, then it is "longest subarray with sum 0".
- 2D range sum → `P[i][j]` plus inclusion–exclusion.
- Many range *updates* → **difference array**: `d[l] += v; d[r+1] -= v`, prefix-sum once at the end.

---

## 4. Hashing and counting

The dumbest, highest-yield optimisation there is: **trade space for the inner loop.**

### Ladder — Two Sum (unsorted)

```
Brute:   all pairs                                     O(n^2)
Optimal: seen = {}; if t - x in seen: hit              O(n)
```

### Ladder — Longest Consecutive Sequence

```
Brute:   sort, then scan runs                          O(n log n)
Optimal: set; only start a walk at x where x-1 not in  O(n)
```

**Unlock:** the "only start at sequence heads" guard is what makes the total work linear — every
element is visited by exactly one chain walk. Without the guard it degrades to O(n²).

**Group Anagrams:** key by `tuple(sorted(word))`, or by a 26-length count tuple for O(n·L).

**Trap:** claiming O(n) while building a `Counter` inside a loop over the same n. That is O(n²).

---

## 5. Binary search

Two distinct uses. Recognising the second is the higher-value skill and the more common miss.

### 5a. Binary search on an index

```python
lo, hi = 0, n            # half-open [lo, hi) kills most off-by-ones
while lo < hi:
    mid = (lo + hi) // 2
    if ok(mid): hi = mid       # mid might be the answer, keep it in range
    else:       lo = mid + 1
return lo                      # first index where ok() is True
```

Use this *one* template for lower_bound, upper_bound and first-true. Never write a three-way
`== target` search — it does not generalise to duplicates or to boundary queries.

**Rotated sorted array:** compare `a[mid]` against a **fixed** endpoint (`a[hi]`), never a moving
one. `a[mid] > a[hi]` → the pivot lies right of mid.

### 5b. Binary search on the answer

**Trigger:** "minimise the maximum", "maximise the minimum", "smallest X such that it's possible",
"kth smallest where counting how many are ≤ v is cheap".

```
Brute:   try every candidate value, test feasibility   O(range · n)
Optimal: binary search the value; ok() is monotone     O(n log range)
```

**Unlock:** you stop *constructing* the answer and start *testing* it. `ok(v)` must be monotone —
once true, true forever after.

```python
def split_array(nums, k):                 # min possible largest subarray sum
    def ok(cap):
        parts, cur = 1, 0
        for x in nums:
            if cur + x > cap: parts, cur = parts + 1, x
            else:             cur += x
        return parts <= k

    lo, hi = max(nums), sum(nums)
    while lo < hi:
        mid = (lo + hi) // 2
        if ok(mid): hi = mid
        else:       lo = mid + 1
    return lo
```

The exact same shape solves Koko Eating Bananas, Capacity to Ship Packages in D Days, Minimum Days
to Make M Bouquets, Kth Smallest Element in a Sorted Matrix, and Kth Smallest Pair Distance.

**Traps:** `lo` and `hi` must bracket a feasible range — derive both bounds out loud. Prove
monotonicity of `ok()`. If you ever write `lo = mid`, you must use `mid = (lo + hi + 1) // 2` or you
infinite-loop.

---

## 6. Sorting, greedy, intervals

**Trigger:** an exchange argument exists — *"moving toward the sorted order never makes the answer
worse."* State the exchange argument; do not just assert that the greedy works.

### Which key to sort by

| Goal | Sort by |
|---|---|
| Max **number** of non-overlapping intervals / meetings | **end** ascending |
| Merge overlapping intervals | **start** ascending |
| Min rooms / max concurrency | start ascending + min-heap of ends, or a ±1 sweep |
| Two competing factors (Course Schedule III) | Sort by the one you want to **freeze**, heap the other |

### Merge intervals

```python
def merge(iv):
    iv.sort()
    out = []
    for s, e in iv:
        if out and s <= out[-1][1]: out[-1][1] = max(out[-1][1], e)
        else:                       out.append([s, e])
    return out
```

### Sweep line — Minimum Meeting Rooms

```
Brute:   for each meeting, count overlaps              O(n^2)
Optimal: sort starts and ends separately, +1/-1 sweep  O(n log n)
```

### Greedy + regret heap — the one that catches people

When you only learn a choice was wrong *after* you have passed it: **take everything, then undo the
worst commitment.**

- *Course Schedule III* — sort by deadline, take every course; when over budget, pop the longest
  course already taken.
- *Furthest Building* — use a ladder on every climb; when you run out, demote the smallest
  ladder-use to bricks.
- *IPO / Maximum Events* — maintain a heap of currently-available options, take the best each step.

**Diagnostic when a greedy breaks:** *is my candidate set too small, or is my move set too small?*
Move set too small → add an undo (a heap or a stack). Candidate set too small → widen what you
consider.

**Fails when** retraction is not legal, because an earlier commitment irreversibly changed the
world. Then it is DP, not greedy.

---

## 7. Monotonic stack and deque

**Trigger:** "next greater", "previous smaller", "span", "largest rectangle", elements that interact
only with their nearest surviving neighbour, or a lexicographically-smallest construction.

### Ladder — Next Greater Element

```
Brute:   for each i, scan right                        O(n^2)
Optimal: stack of indices with decreasing values       O(n)
         each index is pushed once and popped once
```

```python
def next_greater(a):
    res, st = [-1] * len(a), []
    for i, x in enumerate(a):
        while st and a[st[-1]] < x:
            res[st.pop()] = x           # x is the answer for everything it beats
        st.append(i)
    return res
```

### Ladder — Largest Rectangle in Histogram

```
Brute:   for each bar, expand both directions          O(n^2)
Optimal: increasing stack; when a bar pops, its width  O(n)
         is bounded left by the new stack top and
         right by the current index
```

```python
def largest_rect(h):
    st, best = [], 0
    for i, x in enumerate(h + [0]):     # sentinel 0 flushes the stack
        while st and h[st[-1]] >= x:
            ht = h[st.pop()]
            left = st[-1] + 1 if st else 0
            best = max(best, ht * (i - left))
        st.append(i)
    return best
```

Maximal Rectangle in a binary matrix is this, run once per row over a running heights array.

### Ladder — Sliding Window Maximum (monotonic deque)

```
Brute:   max() per window                              O(n·k)
Better:  max-heap with lazy deletion                   O(n log n)
Optimal: deque of decreasing indices                   O(n)
```

```python
from collections import deque

def max_window(a, k):
    dq, out = deque(), []
    for i, x in enumerate(a):
        while dq and a[dq[-1]] <= x: dq.pop()      # x is newer AND bigger -> dominates
        dq.append(i)
        if dq[0] <= i - k: dq.popleft()            # front expired
        if i >= k - 1: out.append(a[dq[0]])
    return out
```

**Unlock:** an element that is both older *and* smaller can never be an answer again, so drop it
permanently.

**Also stack-shaped:** Trapping Rain Water (stack, or two pointers, or prefix-max arrays — know two
of the three), Daily Temperatures, Remove K Digits, Remove Duplicate Letters, Basic Calculator,
Asteroid Collision, Valid Parentheses, Decode String.

---

## 8. Heaps and top-K

**Trigger:** "k largest", "k closest", "median of a stream", "merge k sorted", or repeatedly taking
the current best.

### Ladder — Kth Largest Element

```
Brute:   sort, index                                   O(n log n)
Optimal: min-heap of size k                            O(n log k)
Better:  quickselect                                   O(n) average, O(n^2) worst
Stream:  min-heap of size k is the only option         O(log k) per element
```

Rule to memorise: **kth largest → min-heap of size k** (you pop the smallest). Kth smallest →
max-heap of size k.

### Merge K Sorted Lists

```
Brute:   concatenate and sort                          O(N log N)
Optimal: heap of the k current heads                   O(N log k)
Alt:     pairwise merge, log k rounds                  O(N log k)
```

### Two heaps — median of a data stream

Max-heap over the lower half, min-heap over the upper half, sizes differing by at most 1, rebalance
after every insert. The same structure powers Sliding Window Median (plus lazy deletion).

---

## 9. Linked lists and fast/slow pointers

Three primitives compose into nearly every list problem:

```python
def reverse(head):
    prev = None
    while head:
        head.next, prev, head = prev, head, head.next
    return prev

def middle(head):                     # slow ends at the midpoint
    slow = fast = head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
    return slow

def has_cycle(head):                  # Floyd
    slow = fast = head
    while fast and fast.next:
        slow, fast = slow.next, fast.next.next
        if slow is fast: return True
    return False
```

**Cycle entry point:** after the meeting, reset one pointer to `head` and advance both one step at a
time — they meet at the entry. The same math solves *Find the Duplicate Number* (treat the array as
a functional graph) in O(1) space with a read-only array.

**Always use a dummy head** when the head itself may be removed. Reorder List = middle + reverse +
merge. Palindrome Linked List = the same, in O(1) space.

---

## 10. Trees

Almost everything reduces to: **recurse, and decide what the child returns to the parent.** That
return value *is* the design decision.

```python
def depth(root):                          # returns a scalar upward
    return 0 if not root else 1 + max(depth(root.left), depth(root.right))
```

### Ladder — Diameter of a Binary Tree

```
Brute:   at each node, depth(left) + depth(right)      O(n^2)
Optimal: one post-order pass — return the depth the    O(n)
         parent needs, update a global best en route
```

**Unlock:** the answer *through* a node and the value *needed by* the parent are two different
things. Return the parent's need; accumulate the answer in a side variable. This single move solves
Diameter, Binary Tree Maximum Path Sum, House Robber III, Balanced Binary Tree and Longest Univalue
Path.

### Must-know facts

- **BST inorder traversal is sorted** → validate BST, kth smallest, inorder successor, recover BST.
- Validating a BST needs `(low, high)` bounds passed **down**. A local parent comparison is wrong.
- **LCA:** if root is p or q → root; else recurse both sides; two non-null results → root; otherwise
  the non-null one.
- **BFS by level:** snapshot `for _ in range(len(queue))` at the top of each round.
- Serialize/deserialize: preorder with explicit `#` markers for nulls.
- Build from preorder + inorder: precompute an index map of inorder positions → O(n) not O(n²).

---

## 11. Graphs

**Model it first:** what is a node, what is an edge? Grids are implicit graphs with 4 or 8
neighbours.

| Need | Algorithm | Complexity |
|---|---|---|
| Reachability, components, flood fill | DFS / BFS | O(V+E) |
| Shortest path, **unweighted** | BFS | O(V+E) |
| Shortest path, weights ≥ 0 | Dijkstra with a heap | O(E log V) |
| Shortest path with negative edges | Bellman–Ford | O(V·E) |
| All pairs, small V | Floyd–Warshall | O(V^3) |
| Ordering with dependencies, cycle detection (DAG) | Kahn's topological sort | O(V+E) |
| Dynamic connectivity, MST | Union-Find / Kruskal | ~O(E α) |
| Edge weights only 0 or 1 | 0-1 BFS with a deque | O(V+E) |

### BFS on a grid

```python
from collections import deque

def bfs(grid, starts):
    q, seen, d = deque(starts), set(starts), 0
    while q:
        for _ in range(len(q)):
            r, c = q.popleft()
            for nr, nc in ((r+1, c), (r-1, c), (r, c+1), (r, c-1)):
                if 0 <= nr < len(grid) and 0 <= nc < len(grid[0]) and \
                   (nr, nc) not in seen and grid[nr][nc] != '#':
                    seen.add((nr, nc)); q.append((nr, nc))
        d += 1
    return d
```

**Multi-source BFS** (Rotting Oranges, 01 Matrix, Walls and Gates): push *every* source at distance
0 before you start. That turns an O(sources · V) problem into O(V) — a very common intended
optimisation.

### Topological sort (Kahn)

```python
from collections import deque, defaultdict

def topo(n, edges):
    g, indeg = defaultdict(list), [0] * n
    for u, v in edges:
        g[u].append(v); indeg[v] += 1
    q = deque(i for i in range(n) if indeg[i] == 0)
    out = []
    while q:
        u = q.popleft(); out.append(u)
        for v in g[u]:
            indeg[v] -= 1
            if indeg[v] == 0: q.append(v)
    return out if len(out) == n else []      # short output means there is a cycle
```

Cycle detection in a **directed** graph via DFS needs three colours (white / grey / black). A plain
visited set is only correct for undirected graphs.

### Union-Find

```python
class DSU:
    def __init__(self, n):
        self.p = list(range(n)); self.r = [0] * n

    def find(self, x):
        while self.p[x] != x:
            self.p[x] = self.p[self.p[x]]      # path halving
            x = self.p[x]
        return x

    def union(self, a, b):
        a, b = self.find(a), self.find(b)
        if a == b: return False
        if self.r[a] < self.r[b]: a, b = b, a
        self.p[b] = a
        if self.r[a] == self.r[b]: self.r[a] += 1
        return True
```

Use it when merges arrive **online**, or for Kruskal, Accounts Merge, Redundant Connection, Number
of Islands II.

### Dijkstra

```python
import heapq

def dijkstra(g, src, n):
    dist = [float('inf')] * n; dist[src] = 0
    pq = [(0, src)]
    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]: continue              # stale entry, lazy deletion
        for v, w in g[u]:
            if d + w < dist[v]:
                dist[v] = d + w
                heapq.heappush(pq, (dist[v], v))
    return dist
```

The `if d > dist[u]: continue` line is not optional — it is what keeps this O(E log V).

---

## 12. Backtracking

**Trigger:** enumerate *all* valid configurations. The template is always three moves:
choose → recurse → un-choose.

```python
def subsets(nums):
    res, path = [], []

    def bt(i):
        if i == len(nums):
            res.append(path[:]); return       # copy — path is mutated in place
        bt(i + 1)                             # skip nums[i]
        path.append(nums[i]); bt(i + 1); path.pop()

    bt(0)
    return res
```

### Combination Sum (unbounded reuse)

```python
def comb_sum(cands, target):
    cands.sort(); res, path = [], []

    def bt(start, rem):
        if rem == 0:
            res.append(path[:]); return
        for i in range(start, len(cands)):
            if cands[i] > rem: break                       # prune — needs the sort
            path.append(cands[i])
            bt(i, rem - cands[i])                          # i, not i+1 -> reuse allowed
            path.pop()

    bt(0, target)
    return res
```

**Duplicates in the input:** sort, then inside the loop
`if i > start and cands[i] == cands[i-1]: continue` — this skips duplicate *branches* at the same
depth while still allowing the same value to appear deeper.

**Complexity** is (number of solutions) × (cost to build one). Say it that way rather than
"exponential".

Family: Permutations, Subsets I/II, Combination Sum I/II, Palindrome Partitioning, Word Search,
N-Queens, Letter Combinations, Restore IP Addresses, Sudoku Solver.

**Pruning is the whole game:** sort plus an early `break`, feasibility bounds, and for N-Queens track
occupied columns and diagonals as sets keyed on `r - c` and `r + c` instead of rescanning the board.

---

## 13. Dynamic programming

The most reliable derivation route on the sheet, and it always follows the same four steps:

1. **Write the brute-force recursion with an explicit signature:** `f(i, j, k) -> answer`.
2. **Name the repeated work** — which argument tuples recur?
3. **Memoise on exactly those arguments.** This alone is a correct, interview-acceptable answer.
4. **Flip to bottom-up** and shrink the state if space matters.

*The state is the minimal set of facts about the past that the future actually needs.* Too much
state → TLE or MLE. Too little → wrong answer.

### 13a. Linear DP

**House Robber** — `dp[i] = max(dp[i-1], dp[i-2] + a[i])`, collapsed to two rolling variables:

```python
def rob(a):
    prev = cur = 0
    for x in a:
        prev, cur = cur, max(cur, prev + x)
    return cur
```

**Longest Increasing Subsequence**

```
Brute:   enumerate all subsequences                    O(2^n)
DP:      dp[i] = 1 + max(dp[j]) for j<i, a[j]<a[i]     O(n^2)
Optimal: tails[] + binary search (patience sorting)    O(n log n)
```

```python
import bisect

def lis(a):
    tails = []
    for x in a:
        i = bisect.bisect_left(tails, x)     # bisect_right for the non-decreasing variant
        if i == len(tails): tails.append(x)
        else:               tails[i] = x
    return len(tails)                         # tails is NOT the actual subsequence
```

**Kadane (maximum subarray)** — `cur = max(x, cur + x)`; the decision at each step is "extend or
restart". The circular variant is `max(kadane, total - min_kadane)`, with an all-negative special
case.

### 13b. Knapsack family

```python
# 0/1 knapsack — each item once -> iterate capacity DESCENDING
for w, v in items:
    for c in range(C, w - 1, -1):
        dp[c] = max(dp[c], dp[c - w] + v)

# Unbounded knapsack — reuse allowed -> iterate capacity ASCENDING
for w, v in items:
    for c in range(w, C + 1):
        dp[c] = max(dp[c], dp[c - w] + v)
```

The loop *direction* is the entire difference. Descending guarantees `dp[c - w]` still refers to the
previous item's row.

Also here: Coin Change (min coins → `min`, initialise to infinity), Coin Change II (count ways →
**items in the outer loop, capacity inner**, or you count permutations instead of combinations),
Partition Equal Subset Sum (subset-sum to `total // 2`), Target Sum (reduces to subset-sum).

### 13c. Two-sequence and grid DP — the `(i, j)` table

**Edit Distance**

```python
def edit(a, b):
    m, n = len(a), len(b)
    dp = [[0] * (n + 1) for _ in range(m + 1)]
    for i in range(m + 1): dp[i][0] = i
    for j in range(n + 1): dp[0][j] = j
    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if a[i-1] == b[j-1]:
                dp[i][j] = dp[i-1][j-1]
            else:
                dp[i][j] = 1 + min(dp[i-1][j],       # delete
                                   dp[i][j-1],       # insert
                                   dp[i-1][j-1])     # replace
    return dp[m][n]
```

The same table shape covers LCS, Distinct Subsequences, Regex and Wildcard Matching, Interleaving
String, Unique Paths, Minimum Path Sum. Each row depends only on the previous one → **O(min(m, n))
space** if asked.

### 13d. Interval DP — `n ≤ 500` and "the last thing you do"

Burst Balloons, Matrix Chain Multiplication, Minimum Cost to Cut a Stick, Longest Palindromic
Subsequence.

```python
for length in range(2, n + 1):
    for i in range(n - length + 1):
        j = i + length - 1
        for k in range(i, j):
            dp[i][j] = best(dp[i][j], dp[i][k] + dp[k+1][j] + cost(i, k, j))
```

**Unlock for Burst Balloons:** reason about the **last** balloon burst in a range, not the first —
only then do the two sides become independent. Reversing the decision order is the entire trick.

### 13e. Bitmask DP — `n ≤ 20`

`dp[mask]` = the best result over the set of used items. Travelling Salesman, Assignment Problem,
Partition to K Equal Sum Subsets, Shortest Path Visiting All Nodes. Costs O(2^n · n) or O(2^n · n²).

### 13f. State-machine DP — the stock problems

Track `hold` / `free` (plus `cooldown`, or times k transactions). Draw the transitions as a small
state machine and the whole Best-Time-to-Buy-and-Sell family collapses to four lines.

```python
def max_profit_cooldown(p):
    hold, free, cool = float('-inf'), 0, 0
    for x in p:
        hold, free, cool = max(hold, free - x), max(free, cool), hold + x
    return max(free, cool)
```

---

## 14. Tries

**Trigger:** prefix queries, autocomplete, word dictionaries, or searching many words inside one
text.

```python
class Trie:
    def __init__(self):
        self.kids, self.end = {}, False

    def insert(self, w):
        node = self
        for ch in w:
            node = node.kids.setdefault(ch, Trie())
        node.end = True

    def find(self, w, prefix=False):
        node = self
        for ch in w:
            if ch not in node.kids: return False
            node = node.kids[ch]
        return True if prefix else node.end
```

Insert and search are O(L), independent of dictionary size — that is the win over a hash set, which
cannot answer prefix queries at all. **Word Search II** = trie of the dictionary + DFS over the grid,
pruning the instant the current path leaves the trie. Also: maximum XOR pair, via a binary trie over
32 bits.

---

## 15. Bit manipulation

| Trick | Expression |
|---|---|
| Test bit i | `x >> i & 1` |
| Set / clear / toggle bit i | `x \| 1 << i` · `x & ~(1 << i)` · `x ^ 1 << i` |
| Lowest set bit | `x & -x` |
| Clear the lowest set bit | `x & (x - 1)` |
| Popcount loop | `while x: x &= x - 1; c += 1` |
| Power of two | `x > 0 and x & (x - 1) == 0` |
| Iterate every subset of `mask` | `s = mask; while s: ...; s = (s - 1) & mask` |

**XOR identities:** `x ^ x = 0`, `x ^ 0 = x`, and it is commutative. Single Number becomes a
one-line O(1)-space fold. Single Number II (every element appears three times) → count bits mod 3.
Missing Number → XOR all indices with all values.

---

## 16. Cyclic sort / index-as-hash

**Trigger:** values live in `[1..n]` or `[0..n-1]` **and** O(1) extra space is demanded.

**Unlock:** the array *is* your hash table — value `v` belongs at index `v - 1`.

```python
def first_missing_positive(a):
    n = len(a)
    for i in range(n):
        while 1 <= a[i] <= n and a[a[i] - 1] != a[i]:
            a[a[i] - 1], a[i] = a[i], a[a[i] - 1]      # swap it into place
    for i in range(n):
        if a[i] != i + 1: return i + 1
    return n + 1
```

The nested `while` looks quadratic but each swap places one element permanently, so it is O(n).
Non-destructive variant: mark presence by negating `a[abs(x) - 1]` — that solves Find All Duplicates
and Find All Disappeared Numbers. Find the Duplicate Number with a read-only array → Floyd cycle
detection (§9).

---

## 17. Matrix odds and ends

- **Rotate 90°** = transpose, then reverse each row. Say it, then write it.
- **Set Matrix Zeroes in O(1) space:** use row 0 and column 0 as the marker arrays, with a separate
  flag for column 0 itself.
- **Search a 2D matrix with sorted rows and columns:** start at the top-right; `>` → move left,
  `<` → move down. O(m + n) staircase walk.

---

## The four things to say in every round

Your tracked weak spots, turned into a checklist. Say each one out loud, unprompted.

1. **Clarify semantics before approach.** Sorted? Duplicates? Negatives? Empty input? May I mutate
   the input? What is the tie-break / output-ordering rule?
2. **Convert n into a budget immediately:** *"n ≤ 10^5, so I'm aiming for n log n; O(n²) is 10^10
   and dead."* Then say what that budget **forbids** — that is what selects the pattern.
3. **State the brute force as a function signature, then name the repeated work,** before you
   optimise. Never jump silently to the optimal, and never say "I can't optimise this" without first
   checking (a) auxiliary space and (b) which constraint you have not yet spent.
4. **Dry-run one input you invented** — not the given example — before declaring done. Pick an edge
   case: empty, single element, all equal, all negative.

And when you are stuck: **think out loud.** Silence reads as no progress; a wrong idea narrated
reads as progress, and usually earns you a hint.
