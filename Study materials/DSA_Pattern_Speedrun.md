# DSA Pattern Speedrun

Cram sheet: the ~17 patterns that cover the large majority of interview problems. Every technique
here gets the same three things:

- **Core idea** — one sentence you can say out loud.
- **Ladder** — brute force → each optimisation step, with the observation that unlocks it.
- **Code** — compiling C++.

The observation is what you say in the interview. The code is secondary — but it is here so you can
recall the shape under pressure.

Code is C++17 (`using namespace std;` assumed, headers omitted). Every snippet in this file has been
compiled and run.

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

### C++ overflow rule of thumb

`n ≤ 10^5` with values up to `10^9` → any sum or product is `long long`. Say it while you declare
it. `int` overflow is the single most common silent wrong-answer in a C++ round: prefix sums,
`accumulate(v.begin(), v.end(), 0)` as a binary-search bound, `ht * width` in the histogram problem,
and `(lo + hi) / 2` inside a binary search all need `long long` or a subtraction-based midpoint.

### Phrase → pattern

| You read | Reach for | §|
|---|---|---|
| "contiguous subarray/substring", all values **positive** | Sliding window | 2 |
| "subarray sum = k" with **negatives** | Prefix sum + hash map | 3 |
| "k distinct" / "at most k" over a window | Variable window + counter map | 2 |
| Sorted input, or "pair/triplet summing to X" | Two pointers from both ends | 1 |
| "minimise the maximum" / "maximise the minimum" | Binary search on the answer | 5b |
| "kth smallest/largest", counting `≤ v` is easy | Binary search on value space | 5b |
| "top k" / "k closest" / streaming | Heap of size k | 8 |
| "next greater / previous smaller / span" | Monotonic stack | 7 |
| "max/min of every window of size k" | Monotonic deque | 7 |
| Intervals, meetings, ranges | Sort by start or end, then sweep | 6 |
| "number of ways" / "min cost to reach" | DP | 13 |
| "all combinations / all valid X" | Backtracking | 12 |
| Grid + shortest path, unweighted | BFS | 11 |
| Grid + connected regions | DFS / flood fill / Union-Find | 11 |
| Dependencies, ordering, cycles | Topological sort | 11 |
| Weighted shortest path, non-negative | Dijkstra | 11 |
| Dynamic connectivity, "are these merged" | Union-Find | 11 |
| Prefix matching, word dictionaries | Trie | 14 |
| Values in `[1..n]`, find missing/duplicate | Cyclic sort / index-as-hash | 16 |
| O(1) extra space demanded on an array | Mutate the input as your hash table | 16 |

---

## 1. Two pointers

**Core idea:** maintain two indices whose movement is governed by an invariant strong enough to
prove that the discarded region cannot contain the answer. You throw away a whole row of the
candidate matrix per step, not one cell.

**Trigger:** sorted array, or an independent monotonicity argument.

### 1a. Two Sum on a sorted array

**Problem.** Sorted array `a` and target `t`; return the indices of the two values that sum to `t`.

```
Brute:   for i, for j>i: a[i]+a[j]==t                  O(n^2)
Better:  for i: binary search t-a[i]                   O(n log n)
Optimal: l=0, r=n-1; sum<t -> l++, sum>t -> r--        O(n), O(1) space
```

**Unlock:** if `a[l]+a[r] < t`, then `a[l]` paired with *anything* is still ≤ that sum, so `l` can
never be part of a solution. One comparison eliminates an entire index.

```cpp
pair<int,int> twoSumSorted(const vector<int>& a, long long t) {
    int l = 0, r = (int)a.size() - 1;
    while (l < r) {
        long long s = (long long)a[l] + a[r];
        if (s == t) return {l, r};
        if (s < t) ++l;
        else       --r;
    }
    return {-1, -1};
}
```

### 1b. Container With Most Water

**Problem.** `h[i]` is the height of a vertical line at position `i`. Pick two lines that with the
x-axis hold the most water: `area = (j - i) · min(h[i], h[j])`. Lines in between are ignored — the
water spans over them.

```
Brute:   all pairs, area = (j-i)*min(h[i],h[j])        O(n^2)
Optimal: two ends, always move the SHORTER wall        O(n)
```

**Unlock:** width only shrinks as the pointers close. Moving the taller wall can never increase
`min(h[l], h[r])`, so every pair involving the current shorter wall is *already* at its best width —
that wall can be discarded.

```cpp
int maxArea(const vector<int>& h) {
    int l = 0, r = (int)h.size() - 1, best = 0;
    while (l < r) {
        best = max(best, (r - l) * min(h[l], h[r]));
        if (h[l] < h[r]) ++l;          // move the SHORTER wall
        else             --r;
    }
    return best;
}
```

Note this is a two-pointer walk on **unsorted** data — legal only because of the argument above.
State that argument; do not let it pass as "two pointers usually works here".

### 1c. 3Sum

**Problem.** Return every **unique** triplet summing to zero. The input may contain duplicate
values; the output may not contain duplicate triplets.

```
Brute:   triple loop                                   O(n^3)
Optimal: sort; fix i; two-pointer the suffix           O(n^2)
```

**Unlock:** fixing one element reduces an unsolved problem to a solved one (1a). The single most
reusable move on this sheet.

```cpp
vector<vector<int>> threeSum(vector<int>& a) {
    sort(a.begin(), a.end());
    int n = a.size();
    vector<vector<int>> res;
    for (int i = 0; i < n - 2; ++i) {
        if (i > 0 && a[i] == a[i-1]) continue;          // skip duplicate anchors
        int l = i + 1, r = n - 1;
        while (l < r) {
            long long s = (long long)a[i] + a[l] + a[r];
            if (s < 0) ++l;
            else if (s > 0) --r;
            else {
                res.push_back({a[i], a[l], a[r]});
                while (l < r && a[l] == a[l+1]) ++l;    // AFTER recording the hit
                while (l < r && a[r] == a[r-1]) --r;
                ++l; --r;
            }
        }
    }
    return res;
}
```

**Trap:** duplicate skipping goes *after* you record a hit, not before.

---

## 2. Sliding window

**Core idea:** a window over a contiguous range where the validity metric is **monotone** in the
window's size, so the left edge never needs to move backwards. Each index enters and leaves once →
linear.

**Trigger:** "contiguous", plus a guarantee of monotonicity — all-positive values is the classic
one. Negatives kill it; go to §3.

### 2a. Fixed window

**Problem.** Largest sum of a contiguous subarray of exactly length `k`.

**Unlock:** consecutive windows share all but two elements, so the O(n·k) recompute is waste.

```cpp
long long maxSumFixed(const vector<int>& a, int k) {
    long long s = 0;
    for (int i = 0; i < k; ++i) s += a[i];
    long long best = s;
    for (int i = k; i < (int)a.size(); ++i) {
        s += a[i] - a[i-k];              // add the new, drop the old
        best = max(best, s);
    }
    return best;
}
```

### 2b. Variable window — longest valid

**Problem.** *Longest Substring Without Repeating Characters* — length of the longest substring in
which no character occurs twice.

```
Brute:   all substrings, validate each                 O(n^3)
Better:  incremental validity while extending          O(n^2)
Optimal: expand right always, shrink left while        O(n)
         invalid — each index enters and leaves once
```

```cpp
int longestNoRepeat(const string& s) {
    vector<int> last(128, -1);
    int l = 0, best = 0;
    for (int r = 0; r < (int)s.size(); ++r) {
        if (last[s[r]] >= l) l = last[s[r]] + 1;   // jump, don't crawl
        last[s[r]] = r;
        best = max(best, r - l + 1);
    }
    return best;
}
```

### 2c. Variable window — shortest valid (Minimum Window Substring)

**Problem.** Shortest substring of `s` containing every character of `t`, counting multiplicity.
Order does not matter; extra characters are allowed.

**Unlock:** same window, opposite recording point. Record *inside* the shrink loop, not after
expanding.

```cpp
string minWindow(const string& s, const string& t) {
    vector<int> need(128, 0);
    for (char c : t) ++need[c];
    int missing = t.size(), l = 0, bestLen = INT_MAX, bestL = 0;
    for (int r = 0; r < (int)s.size(); ++r) {
        if (need[s[r]] > 0) --missing;
        --need[s[r]];
        while (missing == 0) {                     // valid -> try to shrink
            if (r - l + 1 < bestLen) { bestLen = r - l + 1; bestL = l; }
            ++need[s[l]];
            if (need[s[l]] > 0) ++missing;
            ++l;
        }
    }
    return bestLen == INT_MAX ? "" : s.substr(bestL, bestLen);
}
```

### 2d. Counting windows, and the "exactly k" trick

**Problem.** *Subarrays with K Different Integers* — count the contiguous subarrays containing
exactly `k` distinct values.

```
Brute:   enumerate every subarray, count distinct      O(n^2)
Optimal: atMost(k) - atMost(k-1)                       O(n)
```

**Unlock:** "exactly k distinct" is **not** monotone, so it has no valid window. "At most k" is.
Subtract two monotone quantities to get the non-monotone one.

The second trick here: **a valid window ending at `r` contributes `r - l + 1` subarrays** — that is
how you *count* rather than measure.

```cpp
long long atMostKDistinct(const vector<int>& a, int k) {
    unordered_map<int,int> cnt;
    long long res = 0;
    int l = 0;
    for (int r = 0; r < (int)a.size(); ++r) {
        if (++cnt[a[r]] == 1) --k;                 // a new distinct value
        while (k < 0)
            if (--cnt[a[l++]] == 0) ++k;
        res += r - l + 1;                          // subarrays ending at r
    }
    return res;
}

long long exactlyKDistinct(const vector<int>& a, int k) {
    return atMostKDistinct(a, k) - atMostKDistinct(a, k - 1);
}
```

**Traps:** the shrink condition is a `while`, never an `if`. Prefer a fixed `vector<int>` over
`unordered_map` when the alphabet is bounded.

---

## 3. Prefix sums + hash map

**Core idea:** `sum(i..j) = P[j] - P[i-1]`, so a question about a *range* becomes a question about
two *points* — and "does a matching point exist" is a hash lookup, not a scan.

**Trigger:** subarray sums with **negative** numbers, or many range-sum queries.

### 3a. Subarray Sum Equals K

**Problem.** Count the contiguous subarrays summing to `k`. Values may be **negative**, which is
what rules out a sliding window.

```
Brute:   all (i,j), sum the inner range                O(n^3)
Better:  running sum in the inner loop                 O(n^2)
Optimal: seen[prefix] counts; res += seen[p - k]       O(n)
```

```cpp
int subarraySum(const vector<int>& a, int k) {
    unordered_map<long long, int> seen;
    seen[0] = 1;                                   // the empty prefix
    long long p = 0;
    int res = 0;
    for (int x : a) {
        p += x;
        auto it = seen.find(p - k);
        if (it != seen.end()) res += it->second;
        ++seen[p];
    }
    return res;
}
```

`seen[0] = 1` seeds the empty prefix — without it you miss every subarray starting at index 0.
`unordered_map::operator[]` value-initialises to `0`, so `++seen[p]` needs no guard — but *reading*
with `[]` also inserts, which is why the lookup uses `find`.

### 3b. Longest subarray with sum k

**Problem.** Length of the longest contiguous subarray summing to `k`.

**Unlock:** counting → store a **count**. Measuring the longest → store the **first** index only.
Same skeleton, different payload.

```cpp
int longestSubarraySumK(const vector<int>& a, int k) {
    unordered_map<long long,int> first;
    first[0] = -1;                                 // empty prefix ends before index 0
    long long p = 0;
    int best = 0;
    for (int i = 0; i < (int)a.size(); ++i) {
        p += a[i];
        auto it = first.find(p - k);
        if (it != first.end()) best = max(best, i - it->second);
        if (!first.count(p)) first[p] = i;         // keep the EARLIEST occurrence
    }
    return best;
}
```

- Sum divisible by k → key on `p % k`, normalising negatives with `((p % k) + k) % k`.
- Equal 0s and 1s → map `0 -> -1`, then it is "longest subarray with sum 0".

### 3c. 2D range sum — inclusion–exclusion

**Problem.** *Range Sum Query 2D* — preprocess a matrix so that any query "sum of the rectangle
from `(r1,c1)` to `(r2,c2)`" is answered in O(1). Many queries, no updates.

```
Brute:   sum the rectangle per query                   O(mn) per query
Optimal: build P once, answer each query in O(1)       O(mn) build, O(1) query
```

**Core idea:** the 1D identity `sum(i..j) = P[j] - P[i-1]` needs *two* points. In 2D it needs
**four**, because a rectangle has four corners.

Define `P[i][j]` = sum of everything **above and to the left** of `(i, j)`, inclusive — one
"upper-left block" per cell. Then the sum of the rectangle from `(r1, c1)` to `(r2, c2)`:

```
    c1      c2                 target = big block
  ┌────┬─────────┐                    − the strip ABOVE it
r1│ A  │    B    │                    − the strip LEFT of it
  ├────┼─────────┤                    + the corner (added back)
  │ C  │ TARGET  │
r2└────┴─────────┘

P[r2][c2]  = A + B + C + TARGET      the whole upper-left block
P[r1-1][c2] = A + B                   everything above the target
P[r2][c1-1] = A + C                   everything left of the target

TARGET = P[r2][c2] - P[r1-1][c2] - P[r2][c1-1] + P[r1-1][c1-1]
                                                  ^ A was subtracted TWICE
```

**Unlock:** subtracting the two strips double-removes their overlap `A`, so you add it back once.
That "+ corner" term is the whole trick, and it's the part people drop.

The build uses the *same* identity in reverse — each cell is its two neighbouring blocks, minus
their shared overlap, plus itself:

```cpp
struct Sum2D {
    vector<vector<long long>> P;                   // 1-indexed: row/col 0 are zeros,
                                                   // so r1-1 and c1-1 need no guard
    Sum2D(const vector<vector<int>>& m)
        : P(m.size() + 1, vector<long long>(m[0].size() + 1, 0)) {
        for (int i = 1; i <= (int)m.size(); ++i)
            for (int j = 1; j <= (int)m[0].size(); ++j)
                P[i][j] = m[i-1][j-1] + P[i-1][j] + P[i][j-1] - P[i-1][j-1];
    }                                              //                  ^ overlap, once

    long long query(int r1, int c1, int r2, int c2) const {      // 0-indexed, inclusive
        ++r1; ++c1; ++r2; ++c2;                    // shift into the padded frame
        return P[r2][c2] - P[r1-1][c2] - P[r2][c1-1] + P[r1-1][c1-1];
    }
};
```

**Pad the table by one row and column.** Without it, every query needs `r1 == 0` and `c1 == 0`
special cases; with it, the zero row and column absorb them.

The same four-corner shape generalises: a 2D **difference** array (§3d) marks `+v` and `-v` at the
four corners of an update rectangle, then two prefix-sum passes apply them all.

### 3d. Difference array — the inverse trick

**Problem.** *Range Addition* — apply `q` updates of the form "add `v` to every index in `[l, r]`",
then read the final array. All updates arrive before any read.

```
Brute:   apply each range update elementwise           O(q·n)
Optimal: mark the two endpoints, prefix-sum once       O(q + n)
```

**Core idea:** define `d[i] = a[i] - a[i-1]`. Then `prefixSum(d) == a` — difference and prefix sum
are inverse operations, like differentiating and integrating.

| | Range **query** | Range **update** |
|---|---|---|
| Prefix sum `P` | O(1) | O(n) — rebuild |
| Difference array `d` | O(n) — rebuild | O(1) |

Prefix sums make *reads* cheap; difference arrays make *writes* cheap. Recognise which side of the
problem you are on.

**Unlock:** adding `v` across `[l, r]` raises a flat plateau. Inside the plateau, consecutive
elements both rose by `v`, so their *difference* is unchanged — only the **edges** move: a step up
at `l`, a step down at `r+1`. That collapses `r - l + 1` writes into exactly 2.

**Worked example** — `n = 5`, all zeros; add 3 to `[0..2]`, 2 to `[1..4]`, 5 to `[3..3]`:

```
                       idx:   0    1    2    3    4  | 5      <- the n+1 dump slot
start                      [  0    0    0    0    0  | 0 ]
+3 on [0..2]  d[0]+=3, d[3]-=3
                           [  3    0    0   -3    0  | 0 ]
+2 on [1..4]  d[1]+=2, d[5]-=2
                           [  3    2    0   -3    0  |-2 ]
+5 on [3..3]  d[3]+=5, d[4]-=5
                           [  3    2    0    2   -5  |-2 ]

prefix-sum once:   3    5    5    7    2
                   ^    ^^^^^^    ^    ^  idx4: only update 2      -> 2
                   |    |         └── idx3: updates 2+3 = 2+5      -> 7
                   |    └── idx1,2: updates 1+2 = 3+2              -> 5
                   └── idx0: update 1 only                         -> 3
```

Three updates cost six writes total, no matter how wide the ranges were.

```cpp
vector<long long> applyRangeUpdates(int n, const vector<array<int,3>>& upd) {
    vector<long long> d(n + 1, 0);                                  // n+1, not n
    for (auto& u : upd) { d[u[0]] += u[2]; d[u[1] + 1] -= u[2]; }   // {l, r, v}
    for (int i = 1; i < n; ++i) d[i] += d[i - 1];                   // one pass -> a
    d.pop_back();
    return d;
}
```

**The `n+1` slot** exists so `d[r+1]` has somewhere to land when `r == n-1`. It is written but never
read — a dump. Sizing `d` as `n` is an out-of-bounds bug that only fires on updates touching the
last index.

**Cost:** `O(q·n)` → `O(q + n)`. With `q = n = 10^5` that is `10^10` → `2×10^5`.

**Where it shows up:** Corporate Flight Bookings, Range Addition, Car Pooling — often over a *time*
axis rather than an index axis. **Minimum Meeting Rooms (§6c) is this trick in disguise:** `+1` at
each start, `-1` at each end, prefix-sum the timeline, take the running max. That is a difference
array with `v = 1`.

**The one constraint — say this out loud:** it only works if you can **batch every update first,
then read**. If updates and queries interleave, each query forces a rebuild and you are back to
`O(q·n)` — that is when you reach for a Fenwick or segment tree. It is the natural follow-up
question, so volunteer it.

---

## 4. Hashing and counting

**Core idea:** the dumbest, highest-yield optimisation there is — trade space for the inner loop.
Any "for each x, search the rest for y" becomes "for each x, look up y".

### 4a. Two Sum (unsorted)

**Problem.** Same as §1a but the array is **unsorted**, so the two-pointer invariant is gone.

```
Brute:   all pairs                                     O(n^2)
Optimal: seen map; if (t - x) in seen: hit             O(n)
```

```cpp
vector<int> twoSum(const vector<int>& a, int t) {
    unordered_map<int,int> seen;                   // value -> index
    for (int i = 0; i < (int)a.size(); ++i) {
        auto it = seen.find(t - a[i]);
        if (it != seen.end()) return {it->second, i};
        seen[a[i]] = i;
    }
    return {};
}
```

### 4b. Longest Consecutive Sequence

**Problem.** Length of the longest run of consecutive integers present in an unsorted array
(`[100,4,200,1,3,2]` → `4`, for `1,2,3,4`). **Required in O(n)** — which bans sorting.

```
Brute:   sort, then scan runs                          O(n log n)
Optimal: hash set; only walk from x where x-1 absent   O(n)
```

**Unlock:** the "only start at sequence heads" guard is what makes the total work linear — every
element is visited by exactly one chain walk. Drop the guard and it degrades to O(n²). *The O(n)
requirement in the statement IS the hint that sorting is banned.*

```cpp
int longestConsecutive(const vector<int>& a) {
    unordered_set<int> s(a.begin(), a.end());
    int best = 0;
    for (int x : s) {
        if (s.count(x - 1)) continue;              // not a sequence head
        int y = x;
        while (s.count(y + 1)) ++y;
        best = max(best, y - x + 1);
    }
    return best;
}
```

### 4c. Group Anagrams — designing the key

**Problem.** Partition a word list so that words which are anagrams of one another land in the same
group.

```
Brute:   compare every pair for anagram-ness           O(n^2 · L)
Better:  key by the sorted string                      O(n · L log L)
Optimal: key by a 26-slot count signature              O(n · L)
```

**Unlock:** "are these equivalent" becomes "do they hash to the same key" once you can name a
**canonical form** for the equivalence class. Choosing the cheapest canonical form is the last step.

```cpp
vector<vector<string>> groupAnagrams(const vector<string>& ws) {
    unordered_map<string, vector<string>> g;
    for (const auto& w : ws) {
        string key(26, 0);                         // count signature, not a sort
        for (char c : w) ++key[c - 'a'];
        g[key].push_back(w);
    }
    vector<vector<string>> res;
    for (auto& kv : g) res.push_back(move(kv.second));
    return res;
}
```

**Traps:** claiming O(n) while rebuilding a frequency map inside a loop over the same n — that is
O(n²). `unordered_map` with a `pair`/`vector` key does not compile without a custom hash; encode the
key into a single integer or use `map` instead of writing one under time pressure.

---

## 5. Binary search

Two distinct uses. Recognising the second is the higher-value skill and the more common miss.

### 5a. Binary search on an index

**Problem shape.** Not one problem — the single template behind lower bound, upper bound, and
"first index where some property holds".

**Core idea:** one template — *find the first index where a monotone predicate turns true.* Every
lower/upper bound question is a rephrasing of that.

```cpp
// lo = 0, hi = n  — half-open [lo, hi) kills most off-by-ones
int firstTrue(int n, const function<bool(int)>& ok) {
    int lo = 0, hi = n;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;      // never (lo + hi) / 2 — overflow
        if (ok(mid)) hi = mid;             // mid might be the answer, keep it in range
        else         lo = mid + 1;
    }
    return lo;                             // == n if ok() is never true
}
```

Never write a three-way `== target` search — it does not generalise to duplicates or boundaries.
The STL gives you the common cases free: `lower_bound` is the first element `>= x`, `upper_bound`
the first `> x`. Say you would use them, then hand-roll if asked.

### 5b. Rotated sorted array

**Problem.** A sorted array rotated at an unknown pivot (`[0,1,2,4,5,6,7]` → `[4,5,6,7,0,1,2]`).
Find a target, or find the minimum, in O(log n).

```
Brute:   linear scan                                   O(n)
Optimal: binary search, anchoring on a FIXED endpoint  O(log n)
```

**Unlock:** at any `mid`, one of the two halves is guaranteed sorted. Identify which, then ask
whether the target lies inside that sorted half — a decidable question.

**Anchor rule:** compare `a[mid]` against a **fixed** endpoint (`a[hi]`), never a moving one.

```cpp
int findMinRotated(const vector<int>& a) {
    int lo = 0, hi = (int)a.size() - 1;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] > a[hi]) lo = mid + 1;      // pivot is strictly right of mid
        else                hi = mid;          // compare against a FIXED end
    }
    return a[lo];
}

int searchRotated(const vector<int>& a, int target) {
    int lo = 0, hi = (int)a.size() - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (a[mid] == target) return mid;
        if (a[lo] <= a[mid]) {                                  // left half sorted
            if (a[lo] <= target && target < a[mid]) hi = mid - 1;
            else                                    lo = mid + 1;
        } else {                                                // right half sorted
            if (a[mid] < target && target <= a[hi]) lo = mid + 1;
            else                                    hi = mid - 1;
        }
    }
    return -1;
}
```

With duplicates the predicate goes uninformative on ties — handle `a[mid] == a[hi]` with `--hi` and
expect to lose the log bound in the worst case. Say that rather than pretending it is still O(log n).

### 5c. Binary search on the answer

**Problem.** *Split Array Largest Sum* — split the array into `k` **contiguous** parts so that the
largest part-sum is as small as possible; return that sum.

**Core idea:** stop *constructing* the answer and start *testing* it. If `ok(v)` is monotone —
once true, true forever — the answer is a boundary you can bisect.

**Trigger:** "minimise the maximum", "maximise the minimum", "smallest X such that it's possible",
"kth smallest where counting how many are ≤ v is cheap".

```
Brute:   try every candidate value, test feasibility   O(range · n)
Optimal: binary search the value; ok() is monotone     O(n log range)
```

```cpp
int splitArray(const vector<int>& nums, int k) {     // min possible largest subarray sum
    auto ok = [&](long long cap) {                   // can we do it in <= k parts?
        int parts = 1;
        long long cur = 0;
        for (int x : nums) {
            if (cur + x > cap) { ++parts; cur = x; }
            else                 cur += x;
        }
        return parts <= k;
    };

    long long lo = *max_element(nums.begin(), nums.end());   // must fit the biggest element
    long long hi = accumulate(nums.begin(), nums.end(), 0LL); // 0LL, not 0
    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (ok(mid)) hi = mid;
        else         lo = mid + 1;
    }
    return (int)lo;
}
```

The identical shape solves Koko Eating Bananas, Capacity to Ship Packages in D Days, Minimum Days to
Make M Bouquets, Kth Smallest Element in a Sorted Matrix and Kth Smallest Pair Distance. Only `ok()`
and the bounds change.

**Traps:** derive both bounds out loud, and prove `ok()` is monotone. If you ever write `lo = mid`,
you must use `mid = lo + (hi - lo + 1) / 2` or you infinite-loop. `accumulate(..., 0)` deduces `int`
and overflows.

---

## 6. Sorting, greedy, intervals

**Core idea:** an exchange argument — *"moving toward the sorted order never makes the answer
worse."* State the exchange argument; do not just assert that the greedy works. The whole decision
is **which key you sort by**, because sorting is how you *freeze* one of two competing factors.

| Goal | Sort by |
|---|---|
| Max **number** of non-overlapping intervals / meetings | **end** ascending |
| Merge overlapping intervals | **start** ascending |
| Min rooms / max concurrency | start ascending + min-heap of ends, or a ±1 sweep |
| Two competing factors (Course Schedule III) | Sort by the one you want to **freeze**, heap the other |

### 6a. Merge intervals — sort by start

**Problem.** Merge all overlapping intervals and return the disjoint set covering the same points.

```
Brute:   repeatedly rescan for any overlapping pair    O(n^2)
Optimal: sort by start; only the last kept interval    O(n log n)
         can possibly overlap the next one
```

```cpp
vector<vector<int>> mergeIntervals(vector<vector<int>>& iv) {
    sort(iv.begin(), iv.end());                    // lexicographic == by start
    vector<vector<int>> out;
    for (auto& in : iv) {
        if (!out.empty() && in[0] <= out.back()[1])
            out.back()[1] = max(out.back()[1], in[1]);
        else
            out.push_back(in);
    }
    return out;
}
```

A custom key is a lambda comparator:
`sort(iv.begin(), iv.end(), [](auto& a, auto& b){ return a[1] < b[1]; });`

### 6b. Max non-overlapping intervals — sort by **end**

**Problem.** *Non-overlapping Intervals* — keep as many intervals as possible with no two
overlapping. (Equivalently: remove the fewest to make the rest disjoint.)

```
Brute:   try every subset, check pairwise disjoint     O(2^n · n)
Optimal: sort by end, take greedily                    O(n log n)
```

**Unlock:** finishing earliest leaves the most room for everything after it. Exchange argument: swap
any optimal solution's first interval for the earliest-ending one — it is still feasible and no
shorter. This is *why* the key is `end`, and why sorting by start is wrong here.

```cpp
int maxNonOverlapping(vector<vector<int>>& iv) {
    sort(iv.begin(), iv.end(),
         [](auto& a, auto& b){ return a[1] < b[1]; });         // by END
    int cnt = 0, end = INT_MIN;
    for (auto& v : iv)
        if (v[0] >= end) { ++cnt; end = v[1]; }
    return cnt;
}
```

### 6c. Minimum Meeting Rooms — heap or sweep

**Problem.** *Meeting Rooms II* — fewest rooms so that no two meetings share a room at the same
time. Equivalently: the maximum number of meetings live at any instant.

```
Brute:   for each meeting, count overlaps              O(n^2)
Heap:    sort by start, min-heap of end times          O(n log n)  <- lead with this
Sweep:   sort starts and ends separately, +1/-1        O(n log n)
```

**Two solutions, and the choice matters.** Know both — the follow-up question decides which one you
needed.

#### Heap — simulate the rooms

**Unlock:** process meetings in arrival order, so the only question per meeting is *"has any room
freed up by now?"* A min-heap of end times puts the soonest-freeing room at `top()`, answering that
in O(1).

```cpp
int minMeetingRoomsHeap(vector<vector<int>> iv) {
    sort(iv.begin(), iv.end());                              // by START
    priority_queue<int, vector<int>, greater<int>> ends;     // MIN-heap of end times
    for (auto& m : iv) {
        if (!ends.empty() && ends.top() <= m[0]) ends.pop(); // earliest room free -> reuse it
        ends.push(m[1]);                                     // occupy a room until m[1]
    }
    return ends.size();                                      // rooms still held = rooms needed
}
```

**`if`, not `while`.** You are seating one meeting, so freeing more than one room buys nothing. And
because each iteration pushes exactly once and pops at most once, the heap size **never decreases** —
so the final size *is* the maximum and no running `best` is needed. Write `while` instead and the
size can drop, so you must then track `best = max(best, ends.size())` yourself. `while` paired with
`return ends.size()` is the bug.

#### Sweep — count concurrency

**Unlock:** you do not care *which* meetings overlap, only *how many* — so decouple the endpoints
from their intervals entirely and sweep a running counter.

```cpp
int minMeetingRooms(vector<vector<int>>& iv) {
    vector<int> st, en;
    for (auto& v : iv) { st.push_back(v[0]); en.push_back(v[1]); }
    sort(st.begin(), st.end());
    sort(en.begin(), en.end());
    int rooms = 0, best = 0, j = 0;
    for (int i = 0; i < (int)st.size(); ++i) {
        while (j < (int)en.size() && en[j] <= st[i]) { --rooms; ++j; }
        ++rooms;
        best = max(best, rooms);
    }
    return best;
}
```

This is a **difference array (§3d) with `v = 1`** — `+1` at each start, `-1` at each end, prefix-sum
the timeline, take the running max.

#### Choosing between them

| | Heap | Sweep |
|---|---|---|
| Mental model | simulate rooms | count concurrency |
| Sorts | 1 (by start) | 2 (starts, ends separately) |
| Knows **which** room | **yes** | no — endpoints are decoupled |
| Generalises to | resource assignment | any "max concurrent" question |

**The deciding row is "which room".** The sweep is fast *precisely because* it throws away the
pairing between a start and its end — which is also why it cannot tell you where anything goes. So
when the follow-up is *"now assign each meeting to a specific room"*, the sweep is a dead end and
the heap extends by carrying the room id:

```cpp
priority_queue<pair<int,int>, vector<pair<int,int>>, greater<>> ends;   // {endTime, roomId}
int nextRoom = 0, r;
// ... for each meeting in start order:
if (!ends.empty() && ends.top().first <= start) { r = ends.top().second; ends.pop(); }
else                                              r = nextRoom++;      // open a new room
ends.push({end, r});
```

**Lead with the heap** — it survives the assignment follow-up and needs one sort. Mention the sweep
as the tighter alternative when only the count is wanted. Offering both *with the reason to prefer
each* scores better than either alone.

### 6d. Greedy + regret heap — the one that catches people

**Problem.** *Course Schedule III* — each course has a duration and a deadline (must **finish** by
day `d`), taken one at a time starting day 1. Maximise how many you take.

**Core idea:** when you only learn a choice was wrong *after* you have passed it, **take everything,
then undo the worst commitment.** A plain greedy has too small a *move set*; adding `pop` fixes it.

```
Brute:   try every subset                              O(2^n)
Greedy:  sort by deadline, take if it fits             WRONG — a long early course
                                                       blocks several short later ones
Optimal: take everything, retract the longest when     O(n log n)
         you go over budget
```

```cpp
int scheduleCourse(vector<vector<int>>& c) {           // {duration, deadline}
    sort(c.begin(), c.end(),
         [](auto& a, auto& b){ return a[1] < b[1]; }); // freeze deadline order
    priority_queue<int> taken;                         // MAX-heap of durations
    long long total = 0;
    for (auto& x : c) {
        taken.push(x[0]);
        total += x[0];
        if (total > x[1]) {                            // over budget -> undo the worst
            total -= taken.top();
            taken.pop();
        }
    }
    return taken.size();
}
```

Same shape, opposite heap direction — spend the scarce resource optimistically, demote the cheapest
use when you run out:

```cpp
int furthestBuilding(const vector<int>& h, int bricks, int ladders) {
    priority_queue<int, vector<int>, greater<int>> used;   // MIN-heap of ladder climbs
    for (int i = 0; i + 1 < (int)h.size(); ++i) {
        int d = h[i+1] - h[i];
        if (d <= 0) continue;
        used.push(d);                                      // optimistically a ladder
        if ((int)used.size() > ladders) {                  // demote the SMALLEST climb
            bricks -= used.top();
            used.pop();
            if (bricks < 0) return i;
        }
    }
    return (int)h.size() - 1;
}
```

**Diagnostic when a greedy breaks:** *is my candidate set too small, or is my move set too small?*
Move set too small → add an undo (a heap or a stack). Candidate set too small → widen what you
consider. **Fails when** retraction is not legal, because an earlier commitment irreversibly changed
the world. Then it is DP, not greedy.

**C++ note:** `priority_queue<int>` is a **max**-heap. Min-heap is
`priority_queue<int, vector<int>, greater<int>>`. Say which one you want before you type it.

---

## 7. Monotonic stack and deque

**Core idea:** keep only the elements that could still be an answer. If an element is dominated —
older *and* worse — drop it forever. Each index is pushed and popped once, so the total is linear
despite the nested `while`.

**Trigger:** "next greater", "previous smaller", "span", "largest rectangle", elements interacting
only with their nearest surviving neighbour, or a lexicographically-smallest construction.

### 7a. Next Greater Element

**Problem.** For each element, the first strictly greater value to its right, or `-1` if none.

```
Brute:   for each i, scan right                        O(n^2)
Optimal: stack of indices, values non-increasing       O(n)
```

**Unlock:** invert the question. Instead of "what is the answer for `i`", ask "who is `a[i]` the
answer *for*" — everything it pops.

```cpp
vector<int> nextGreater(const vector<int>& a) {
    int n = a.size();
    vector<int> res(n, -1);
    stack<int> st;                                 // indices; values NON-INCREASING
    for (int i = 0; i < n; ++i) {                  // (equal values are not popped)
        while (!st.empty() && a[st.top()] < a[i]) {   // strict < : see below
            res[st.top()] = a[i];                  // a[i] answers everything it beats
            st.pop();
        }
        st.push(i);
    }
    return res;
}
```

#### The right-to-left form — equally valid, often more intuitive

Same problem, opposite sweep. Walk **backwards** and let the stack hold the candidates that are
still viable answers for whatever comes next; the top is your answer directly, no assignment on pop.

```cpp
vector<int> nextGreaterRight(const vector<int>& nums) {
    int n = nums.size();
    vector<int> result(n, -1);
    stack<int> st;                                 // VALUES to my right that can still be an answer
    for (int i = n - 1; i >= 0; --i) {
        while (!st.empty() && st.top() <= nums[i]) st.pop();   // can't beat me -> useless to all
        if (!st.empty()) result[i] = st.top();                 // nearest survivor IS the answer
        st.push(nums[i]);
    }
    return result;
}
```

Verified identical to the forward version on 300k random arrays with heavy duplicates.

**Why the pop is safe:** anything `≤ nums[i]` is shadowed forever. Any element further left that
would have accepted it will accept `nums[i]` instead — and `nums[i]` is *nearer*. Dominated, so
discard.

**Which to use.** They differ only in what the pop *means*:

| | Forward (L→R) | Backward (R→L) |
|---|---|---|
| Stack holds | indices whose answer is still unknown | values still viable as an answer |
| Pop means | "found your answer" — assign on pop | "you're dominated" — discard |
| Answer read at | pop time | `st.top()` after popping |
| Needed for §7b / §7c | **yes** | no |

Prefer whichever you can write without thinking — for plain Next Greater they are interchangeable.
**But the forward form is not optional for §7b (Largest Rectangle) and §7c (Trapping Rain Water):**
there the pop is the moment *both* boundaries of a bar become known at once, and that only happens
sweeping forward. Backward would need two passes.

**Store indices, not values, the moment you need a distance.** Daily Temperatures wants
`j - i`, which values cannot give you — in either direction:

```cpp
while (!st.empty() && t[st.top()] <= t[i]) st.pop();   // stack of INDICES
if (!st.empty()) r[i] = st.top() - i;
st.push(i);
```

#### The pop guard's strictness is the spec

`<` in the forward version pops only what `a[i]` *strictly* beats, so equal values survive — the
invariant is **non-increasing**, not decreasing. Flip it and you silently solve a different problem,
"next greater **or equal**":

```
a = [5, 5, 5]     forward with <    ->  [-1, -1, -1]   correct for "strictly greater"
                  forward with <=   ->  [ 5,  5, -1]   answers a different question
```

**The strictness flips with the direction** — the two forms guard on opposite operators to mean the
same thing, because one pops *answers* and the other discards *candidates*:

| You want | Forward guard | Backward guard |
|---|---|---|
| next **strictly** greater | `a[st.top()] < a[i]` | `st.top() <= nums[i]` |
| next greater **or equal** | `a[st.top()] <= a[i]` | `st.top() < nums[i]` |

Pick it from the wording and say which one you are building. Every monotonic-stack problem turns on
this one character, and the sheet's neighbours disagree on purpose: §7b pops on `>=` (strictly
increasing stack) and §7d pops on `<=` (strictly decreasing deque), because each needs the opposite
tie-breaking.

### 7b. Largest Rectangle in Histogram

**Problem.** Largest axis-aligned rectangle that fits entirely under a histogram of bar heights.

```
Brute:   for each bar, expand both directions          O(n^2)
Better:  precompute previous-smaller / next-smaller     O(n) + O(n) space
Optimal: one increasing stack does both at once        O(n)
```

**Unlock:** when a bar is popped, both of its boundaries are known *at that instant* — the new stack
top is its left limit, the current index is its right limit.

```cpp
long long largestRectangle(vector<int> h) {
    h.push_back(0);                                // sentinel flushes the stack
    stack<int> st;
    long long best = 0;
    for (int i = 0; i < (int)h.size(); ++i) {
        while (!st.empty() && h[st.top()] >= h[i]) {
            long long ht = h[st.top()];
            st.pop();
            int left = st.empty() ? 0 : st.top() + 1;
            best = max(best, ht * (i - left));      // ht is long long -> no overflow
        }
        st.push(i);
    }
    return best;
}
```

Maximal Rectangle in a binary matrix is this, run once per row over a running heights array.

### 7c. Trapping Rain Water

**Problem.** Same height array as §1b — but here the bars **do** hold water. How much collects in
the dips after rain?

```
Brute:   for each bar, scan both ways for the walls    O(n^2)
Better:  prefix-max and suffix-max arrays              O(n) time, O(n) space
Optimal: two pointers carrying both running maxima     O(n) time, O(1) space
```

**Unlock:** water above bar `i` is `min(maxLeft, maxRight) - h[i]`. You do not need both maxima
exactly — you only need to know **which one is smaller**, and the shorter side already tells you
that. So process whichever side is currently shorter.

```cpp
long long trap(const vector<int>& h) {
    int l = 0, r = (int)h.size() - 1, lmax = 0, rmax = 0;
    long long res = 0;
    while (l < r) {
        if (h[l] < h[r]) {                     // left wall is the binding one
            lmax = max(lmax, h[l]);
            res += lmax - h[l];
            ++l;
        } else {
            rmax = max(rmax, h[r]);
            res += rmax - h[r];
            --r;
        }
    }
    return res;
}
```

Know two of the three approaches — the stack version also works and is worth naming.

### 7d. Monotonic deque — Sliding Window Maximum

**Problem.** The maximum of every contiguous window of size `k`, as the window slides across.

```
Brute:   max() per window                              O(n·k)
Better:  max-heap with lazy deletion                   O(n log n)
Optimal: deque of decreasing indices                   O(n)
```

**Unlock:** an element that is both older *and* smaller can never be an answer again. A heap cannot
express "expire from the front"; a deque can, because it has two ends.

```cpp
vector<int> maxWindow(const vector<int>& a, int k) {
    deque<int> dq;                                 // indices, values decreasing
    vector<int> out;
    for (int i = 0; i < (int)a.size(); ++i) {
        while (!dq.empty() && a[dq.back()] <= a[i]) dq.pop_back();  // newer AND bigger
        dq.push_back(i);
        if (dq.front() <= i - k) dq.pop_front();   // front expired
        if (i >= k - 1) out.push_back(a[dq.front()]);
    }
    return out;
}
```

**Also stack-shaped:** Remove K Digits, Remove Duplicate Letters, Basic Calculator, Asteroid
Collision, Valid Parentheses, Decode String. In all of them the pop guard is a problem-specific
feasibility question — *getting the guard right is the correctness argument.*

**C++ note:** `std::stack` cannot be iterated. If you need to inspect below the top, use a
`vector<int>` as your stack (`back()` / `pop_back()`).

---

## 8. Heaps and top-K

**Core idea:** you rarely need the whole order — only the boundary between "in" and "out". A heap of
size k maintains exactly that boundary for O(log k) per element, and works on a stream where sorting
cannot.

### 8a. Kth Largest Element

**Problem.** The kth largest value by **rank**, not by distinct value — in `[3,3,1]`, the 2nd
largest is `3`.

```
Brute:   sort, index                                   O(n log n)
Optimal: min-heap of size k                            O(n log k), O(k) space
Better:  nth_element (quickselect), one-shot only      O(n) average
Stream:  min-heap of size k is the ONLY option         O(log k) per element
```

Rule to memorise: **kth largest → min-heap of size k** (you pop the smallest, so the survivors are
the k biggest). Kth smallest → max-heap of size k.

```cpp
int kthLargest(const vector<int>& a, int k) {
    priority_queue<int, vector<int>, greater<int>> pq;     // MIN-heap
    for (int x : a) {
        pq.push(x);
        if ((int)pq.size() > k) pq.pop();                  // evict the smallest
    }
    return pq.top();
}

int kthLargestFast(vector<int> a, int k) {                 // one-shot, mutable input
    nth_element(a.begin(), a.begin() + (k - 1), a.end(), greater<int>());
    return a[k - 1];
}
```

### 8b. Merge K Sorted Lists

**Problem.** Merge `k` sorted linked lists into one sorted list. `N` = total nodes across all lists.

```
Brute:   concatenate and sort                          O(N log N)
Optimal: heap of the k current heads                   O(N log k)
Alt:     pairwise merge, log k rounds                  O(N log k)
```

**Unlock:** the global minimum is always among the k current heads, so the frontier you must track
is size k, not size N.

```cpp
ListNode* mergeKLists(vector<ListNode*>& lists) {
    auto cmp = [](ListNode* a, ListNode* b){ return a->val > b->val; };  // -> min-heap
    priority_queue<ListNode*, vector<ListNode*>, decltype(cmp)> pq(cmp);
    for (auto* l : lists) if (l) pq.push(l);

    ListNode dummy(0), *tail = &dummy;
    while (!pq.empty()) {
        ListNode* n = pq.top(); pq.pop();
        tail->next = n; tail = n;
        if (n->next) pq.push(n->next);
    }
    tail->next = nullptr;
    return dummy.next;
}
```

The comparator is **inverted**: `priority_queue` puts the largest-by-comparator under `top()`, so
`a->val > b->val` yields a min-heap.

### 8c. Two heaps — median of a data stream

**Problem.** *Find Median from Data Stream* — support `add(x)` on an unbounded stream and
`median()` at any point.

```
Brute:   keep a sorted vector, insert each element     O(n) per add
Better:  sort on every query                           O(n log n) per query
Optimal: max-heap of the low half + min-heap of the    O(log n) add, O(1) query
         high half, sizes balanced within 1
```

**Unlock:** the median only depends on the *boundary* between the two halves. You never need the
halves themselves ordered — only their extremes, which is exactly what a heap gives you.

```cpp
struct MedianFinder {
    priority_queue<int> lo;                                // max-heap, lower half
    priority_queue<int, vector<int>, greater<int>> hi;     // min-heap, upper half

    void add(int x) {
        lo.push(x);
        hi.push(lo.top()); lo.pop();                       // funnel through to order it
        if (hi.size() > lo.size()) { lo.push(hi.top()); hi.pop(); }
    }

    double median() const {
        return lo.size() > hi.size() ? lo.top() : (lo.top() + hi.top()) / 2.0;
    }
};
```

The push-then-funnel trick avoids every comparison branch you would otherwise write. The same
structure powers Sliding Window Median (plus lazy deletion, or a `multiset` with a mid-iterator).

---

## 9. Linked lists and fast/slow pointers

**Core idea:** you cannot index a list, so you manufacture position with a second pointer moving at
a different speed. Three primitives compose into nearly every list problem.

**Problems these three cover.** *Reverse Linked List*; *Middle of the Linked List*; *Linked List
Cycle* (does it loop?). Everything below composes them.

```cpp
ListNode* reverseList(ListNode* head) {
    ListNode* prev = nullptr;
    while (head) {
        ListNode* nxt = head->next;
        head->next = prev;
        prev = head;
        head = nxt;
    }
    return prev;
}

ListNode* middle(ListNode* head) {                 // slow ends at the midpoint
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) { slow = slow->next; fast = fast->next->next; }
    return slow;
}

bool hasCycle(ListNode* head) {                    // Floyd
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) return true;
    }
    return false;
}
```

### 9a. Cycle entry point

**Problem.** *Linked List Cycle II* — a list may end in a cycle; return the node where the cycle
begins, in O(1) space.

```
Brute:   hash set of visited nodes                     O(n) time, O(n) space
Optimal: Floyd, then reset one pointer to head         O(n) time, O(1) space
```

**Unlock:** at the meeting point, the distance from `head` to the entry equals the distance from the
meeting point to the entry. So walking both at speed 1 lands them together on the entry.

```cpp
ListNode* cycleStart(ListNode* head) {
    ListNode *slow = head, *fast = head;
    while (fast && fast->next) {
        slow = slow->next;
        fast = fast->next->next;
        if (slow == fast) {                        // meeting point
            slow = head;
            while (slow != fast) { slow = slow->next; fast = fast->next; }
            return slow;                           // the entry
        }
    }
    return nullptr;
}
```

### 9b. Find the Duplicate Number — the same math on an array

**Problem.** `n + 1` values in the range `[1..n]`, exactly one repeated. Find it **without
modifying the array** and in O(1) space.

```
Brute:   sort, or a seen-set                           O(n log n) / O(n) space
Optimal: treat i -> a[i] as a functional graph;        O(n) time, O(1) space,
         the duplicate is the cycle entry              input untouched
```

**Unlock:** "values in `[1..n]` with `n+1` slots" means `i -> a[i]` is a function with a guaranteed
collision — and a collision in a functional graph *is* a cycle entry.

```cpp
int findDuplicate(const vector<int>& a) {
    int slow = a[0], fast = a[a[0]];
    while (slow != fast) { slow = a[slow]; fast = a[a[fast]]; }
    slow = 0;
    while (slow != fast) { slow = a[slow]; fast = a[fast]; }
    return slow;
}
```

**Always use a dummy head** when the head itself may be removed — `ListNode dummy(0); dummy.next =
head;` then return `dummy.next`. Reorder List = middle + reverse + merge. Palindrome Linked List =
the same, in O(1) space.

---

## 10. Trees

**Core idea:** recurse, and **decide what the child returns to the parent.** That return value *is*
the design decision, and it is frequently not the answer you are asked for.

```cpp
int depth(TreeNode* root) {                        // returns a scalar upward
    if (!root) return 0;
    return 1 + max(depth(root->left), depth(root->right));
}
```

### 10a. Diameter — the return-value split

**Problem.** Longest path (counted in edges) between any two nodes. It need not pass through the
root — that is the whole difficulty.

```
Brute:   at each node, depth(left) + depth(right)      O(n^2)
Optimal: one post-order pass — return what the PARENT  O(n)
         needs, accumulate the ANSWER on the side
```

**Unlock:** the answer *through* a node (`l + r`) and the value *needed by* the parent
(`1 + max(l, r)`) are different quantities. Trying to return one value for both is what forces the
quadratic version.

```cpp
int diameterBest = 0;
int diameterDfs(TreeNode* node) {
    if (!node) return 0;
    int l = diameterDfs(node->left), r = diameterDfs(node->right);
    diameterBest = max(diameterBest, l + r);       // the answer THROUGH this node
    return 1 + max(l, r);                          // what the PARENT needs
}
```

This one move also solves Binary Tree Maximum Path Sum, House Robber III, Balanced Binary Tree and
Longest Univalue Path. Max Path Sum adds one twist — clamp negative arms to zero:

```cpp
long long maxPathBest = LLONG_MIN;
int maxPathGain(TreeNode* node) {
    if (!node) return 0;
    int l = max(maxPathGain(node->left), 0);       // a negative arm is worth skipping
    int r = max(maxPathGain(node->right), 0);
    maxPathBest = max(maxPathBest, (long long)node->val + l + r);
    return node->val + max(l, r);
}
```

Prefer passing the accumulator by reference (`int& best`) or as a member over a true global.

### 10b. Validate BST — bounds go **down**, not up

**Problem.** Is every node greater than **everything** in its left subtree and less than everything
in its right? (Not just greater than its immediate left child.)

```
Brute:   for each node, scan its whole subtree          O(n^2)
Wrong:   compare each node only to its parent           FAILS on a grandchild violation
Optimal: pass an open interval (lo, hi) downward        O(n)
```

**Unlock:** BST-ness is not a local property. What a subtree needs to know from above is the range
it is allowed to occupy — so the information flows **down**, not up.

```cpp
bool validBST(TreeNode* n, long long lo = LLONG_MIN, long long hi = LLONG_MAX) {
    if (!n) return true;
    if (n->val <= lo || n->val >= hi) return false;
    return validBST(n->left, lo, n->val) && validBST(n->right, n->val, hi);
}
```

The bounds must be `long long` — node values can legitimately be `INT_MIN` / `INT_MAX`.

**BST inorder traversal is sorted** — that single fact solves kth smallest, inorder successor,
recover BST, and "convert to sorted list".

### 10c. Lowest Common Ancestor

**Problem.** Deepest node having both `p` and `q` as descendants, where a node counts as a
descendant of itself.

**Unlock:** you do not need paths. Ask each subtree "did you find either target?" — the node where
both answers come back non-null is the LCA.

```cpp
TreeNode* lca(TreeNode* r, TreeNode* p, TreeNode* q) {
    if (!r || r == p || r == q) return r;
    TreeNode* l  = lca(r->left,  p, q);
    TreeNode* rr = lca(r->right, p, q);
    if (l && rr) return r;                         // targets split here -> this is the LCA
    return l ? l : rr;
}
```

### 10d. BFS by level

**Problem.** Return node values grouped level by level, top down.

**Unlock:** snapshot the queue size at the top of each round — that count *is* the level boundary.

```cpp
vector<vector<int>> levelOrder(TreeNode* root) {
    vector<vector<int>> res;
    if (!root) return res;
    queue<TreeNode*> q;
    q.push(root);
    while (!q.empty()) {
        int sz = q.size();                         // snapshot BEFORE the inner loop
        vector<int> level;
        while (sz--) {
            TreeNode* n = q.front(); q.pop();
            level.push_back(n->val);
            if (n->left)  q.push(n->left);
            if (n->right) q.push(n->right);
        }
        res.push_back(move(level));
    }
    return res;
}
```

Also worth having ready: serialize/deserialize via preorder with explicit `#` null markers, and
building from preorder + inorder using an `unordered_map<int,int>` of inorder positions → O(n) not
O(n²).

---

## 11. Graphs

**Core idea:** model it first — what is a node, what is an edge? Once named, the algorithm is a
lookup. Grids are implicit graphs with 4 or 8 neighbours.

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

### 11a. Flood fill / connected components

**Problem.** *Number of Islands* — count maximal groups of `'1'` cells connected horizontally or
vertically.

```
Brute:   for each cell, BFS the whole grid to test     O((mn)^2)
Optimal: one DFS per unvisited cell, sinking as        O(mn)
         you go — every cell is entered once
```

```cpp
void sink(vector<vector<char>>& g, int r, int c) {
    if (r < 0 || r >= (int)g.size() || c < 0 || c >= (int)g[0].size()) return;
    if (g[r][c] != '1') return;
    g[r][c] = '0';                                 // mark visited in place
    sink(g, r+1, c); sink(g, r-1, c); sink(g, r, c+1); sink(g, r, c-1);
}

int numIslands(vector<vector<char>>& g) {
    int cnt = 0;
    for (int r = 0; r < (int)g.size(); ++r)
        for (int c = 0; c < (int)g[0].size(); ++c)
            if (g[r][c] == '1') { ++cnt; sink(g, r, c); }
    return cnt;
}
```

Ask whether you may mutate the input — if yes, the grid *is* your visited array and you save O(mn)
space. If not, carry a `vector<vector<bool>>`.

### 11b. BFS on a grid, and multi-source BFS

**Problem.** Fewest steps through an open grid. Multi-source version — *Rotting Oranges*: every
rotten orange infects its neighbours each minute; how many minutes until none are fresh?

```
Brute:   DFS every path, keep the shortest             exponential
Better:  BFS from each source separately               O(sources · V)
Optimal: push ALL sources at distance 0, one BFS       O(V)
```

**Unlock:** BFS explores in distance order, so the first time you reach a cell is its shortest
distance. Seeding every source at once makes the frontier compute `min over all sources` for free.

```cpp
int bfsGrid(const vector<vector<char>>& g, const vector<pair<int,int>>& starts) {
    int m = g.size(), n = g[0].size();
    vector<vector<bool>> seen(m, vector<bool>(n, false));
    deque<pair<int,int>> q;
    for (auto& s : starts) { q.push_back(s); seen[s.first][s.second] = true; }

    const int dr[] = {1, -1, 0, 0}, dc[] = {0, 0, 1, -1};
    int d = 0;
    while (!q.empty()) {
        int sz = q.size();
        while (sz--) {
            auto [r, c] = q.front(); q.pop_front();
            for (int k = 0; k < 4; ++k) {
                int nr = r + dr[k], nc = c + dc[k];
                if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
                if (seen[nr][nc] || g[nr][nc] == '#') continue;
                seen[nr][nc] = true;
                q.push_back({nr, nc});
            }
        }
        if (!q.empty()) ++d;      // only count a level that actually has nodes
    }
    return d;                     // max distance from the nearest source
}
```

The `dr[]`/`dc[]` direction arrays are worth muscle memory — they remove four copy-pasted neighbour
lines and the bugs that hide in them.

**The off-by-one to watch:** a bare `++d` at the bottom of the outer loop counts *levels*, so it
returns `maxDistance + 1`. The `if (!q.empty())` guard is what makes `d` a distance. Rotting Oranges
wants the distance. Say which one you are returning.

### 11c. Topological sort (Kahn)

**Problem.** Order nodes so every edge points forward. *Course Schedule* — given prerequisite
pairs, can all courses be finished, and in what order?

```
Brute:   repeatedly scan for a zero-indegree node      O(V^2)
Optimal: queue the zero-indegree frontier, decrement   O(V+E)
         indegrees as you remove nodes
```

**Unlock:** a node becomes available the instant its last prerequisite is removed — so decrement on
removal and enqueue at zero, instead of rescanning.

```cpp
vector<int> topo(int n, const vector<pair<int,int>>& edges) {
    vector<vector<int>> g(n);
    vector<int> indeg(n, 0);
    for (auto& e : edges) { g[e.first].push_back(e.second); ++indeg[e.second]; }

    queue<int> q;
    for (int i = 0; i < n; ++i) if (indeg[i] == 0) q.push(i);

    vector<int> out;
    while (!q.empty()) {
        int u = q.front(); q.pop();
        out.push_back(u);
        for (int v : g[u]) if (--indeg[v] == 0) q.push(v);
    }
    return (int)out.size() == n ? out : vector<int>{};   // short output => a cycle
}
```

A short output *is* the cycle detector — Course Schedule I is `topo(...).size() == n`.

### 11d. Directed cycle detection via DFS — three colours

**Problem.** Does a **directed** graph contain a cycle?

**Unlock:** in a directed graph, meeting a visited node is not a cycle — it may be a cross edge to a
finished branch. Only an edge back to a node **on the current recursion stack** is a cycle, so
`visited` must distinguish "in progress" from "done".

```cpp
bool hasCycleDFS(int u, const vector<vector<int>>& g, vector<int>& color) {
    color[u] = 1;                                  // 1 = on the current stack
    for (int v : g[u]) {
        if (color[v] == 1) return true;            // back edge -> cycle
        if (color[v] == 0 && hasCycleDFS(v, g, color)) return true;
    }
    color[u] = 2;                                  // 2 = fully explored
    return false;
}
```

A plain boolean `visited` is only correct for **undirected** graphs.

### 11e. Union-Find

**Problem.** Support "merge these two" and "are these connected?" **interleaved**, arriving one at
a time.

```
Brute:   BFS after every merge to test connectivity    O(q · (V+E))
Optimal: DSU with path compression + union by rank     ~O(q · α)
```

**Unlock:** use it when merges arrive **online** and you only ever ask "same component?". If the
whole graph is known up front, plain DFS is simpler — do not reach for DSU reflexively.

```cpp
struct DSU {
    vector<int> p, r;
    DSU(int n) : p(n), r(n, 0) { iota(p.begin(), p.end(), 0); }

    int find(int x) {
        while (p[x] != x) { p[x] = p[p[x]]; x = p[x]; }   // path halving
        return x;
    }

    bool unite(int a, int b) {                            // `union` is a keyword
        a = find(a); b = find(b);
        if (a == b) return false;                         // already together
        if (r[a] < r[b]) swap(a, b);
        p[b] = a;
        if (r[a] == r[b]) ++r[a];
        return true;
    }
};
```

`unite` returning `false` is the answer to Redundant Connection and the filter in Kruskal's MST.
Also: Accounts Merge, Number of Islands II.

### 11f. Dijkstra

**Problem.** Shortest path from a source, with non-negative edge weights.

```
Brute:   BFS ignoring weights                          WRONG on weighted graphs
Better:  scan all V for the nearest unsettled node     O(V^2)
Optimal: min-heap frontier + lazy deletion             O(E log V)
```

**Unlock:** settle nodes in increasing distance order; the first time you pop a node, its distance
is final. Requires non-negative weights — a negative edge could improve a settled node.

```cpp
vector<long long> dijkstra(const vector<vector<pair<int,int>>>& g, int src, int n) {
    const long long INF = LLONG_MAX / 4;           // /4 so d + w cannot overflow
    vector<long long> dist(n, INF);
    dist[src] = 0;

    priority_queue<pair<long long,int>,
                   vector<pair<long long,int>>,
                   greater<>> pq;                  // MIN-heap on distance
    pq.push({0, src});

    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;                 // stale entry, lazy deletion
        for (auto& [v, w] : g[u])
            if (d + w < dist[v]) {
                dist[v] = d + w;
                pq.push({dist[v], v});
            }
    }
    return dist;
}
```

The `if (d > dist[u]) continue;` line is not optional — it is what keeps this O(E log V).

### 11g. 0-1 BFS

**Problem.** Shortest path where every edge weight is 0 or 1 — e.g. a grid where some moves are
free and others cost one.

**Unlock:** when weights are only 0 or 1, a deque replaces the heap — a 0-edge keeps the same
distance so it goes to the **front**, a 1-edge goes to the back. Drops the log factor entirely.

```cpp
vector<int> zeroOneBFS(const vector<vector<pair<int,int>>>& g, int src, int n) {
    vector<int> dist(n, INT_MAX);
    dist[src] = 0;
    deque<int> dq;
    dq.push_back(src);
    while (!dq.empty()) {
        int u = dq.front(); dq.pop_front();
        for (auto& [v, w] : g[u])
            if (dist[u] + w < dist[v]) {
                dist[v] = dist[u] + w;
                if (w == 0) dq.push_front(v);      // same layer
                else        dq.push_back(v);       // next layer
            }
    }
    return dist;
}
```

---

## 12. Backtracking

**Core idea:** choose → recurse → **un-choose**. The template never changes; the problem is entirely
in the pruning and in how you suppress duplicate branches.

**Complexity** is (number of solutions) × (cost to build one). Say it that way rather than
"exponential".

### 12a. Subsets

**Problem.** Every subset of a set of distinct integers (the power set) — `2^n` of them.

```
Brute:   enumerate 0..2^n-1, decode each bitmask       O(2^n · n)
Optimal: recursion sharing the prefix across branches  O(2^n · n) but with
                                                       no re-decoding, and prunable
```

```cpp
void btSubsets(int i, vector<int>& nums, vector<int>& path,
               vector<vector<int>>& res) {
    if (i == (int)nums.size()) { res.push_back(path); return; }   // copies path
    btSubsets(i + 1, nums, path, res);                            // skip nums[i]
    path.push_back(nums[i]);
    btSubsets(i + 1, nums, path, res);                            // take nums[i]
    path.pop_back();                                              // un-choose
}
```

`res.push_back(path)` copies, which is what you want — `path` keeps mutating. Pass `path` and `res`
by **reference**; by value turns this into an exponential-copy disaster and is a common C++-specific
bug.

### 12b. Permutations

**Problem.** Every ordering of the input — `n!` of them.

**Unlock:** swap-in-place rather than tracking a `used[]` array — position `i` gets each remaining
candidate in turn, and the swap back restores the state.

```cpp
void btPermute(int i, vector<int>& a, vector<vector<int>>& res) {
    if (i == (int)a.size()) { res.push_back(a); return; }
    for (int j = i; j < (int)a.size(); ++j) {
        swap(a[i], a[j]);
        btPermute(i + 1, a, res);
        swap(a[i], a[j]);                          // un-choose
    }
}
```

### 12c. Combination Sum — pruning is the whole game

**Problem.** Every multiset of candidates summing to `target`; each candidate may be reused
unlimited times.

```
Brute:   enumerate all multisets, filter by sum        astronomically wasteful
Optimal: sort, then break the moment c[i] > rem        prunes entire subtrees
```

```cpp
void btCombSum(int start, int rem, vector<int>& c, vector<int>& path,
               vector<vector<int>>& res) {
    if (rem == 0) { res.push_back(path); return; }
    for (int i = start; i < (int)c.size(); ++i) {
        if (c[i] > rem) break;                     // prune — needs the sort
        path.push_back(c[i]);
        btCombSum(i, rem - c[i], c, path, res);    // i, not i+1 -> reuse allowed
        path.pop_back();
    }
}

vector<vector<int>> combinationSum(vector<int>& c, int target) {
    sort(c.begin(), c.end());
    vector<int> path;
    vector<vector<int>> res;
    btCombSum(0, target, c, path, res);
    return res;
}
```

**Duplicates in the input:** sort, then inside the loop
`if (i > start && c[i] == c[i-1]) continue;` — that skips duplicate *branches* at the same depth
while still allowing the same value to appear deeper.

### 12d. N-Queens — carry the constraint, don't rescan

**Problem.** Place `n` queens on an `n × n` board with no two sharing a row, column or diagonal;
count (or list) the arrangements.

**Unlock:** three boolean arrays keyed on `col`, `row - col + n` and `row + col` make every conflict
check O(1). Rescanning the board per placement is the brute force.

```cpp
int solveNQueens(int n, int row, vector<char>& col,
                 vector<char>& d1, vector<char>& d2) {
    if (row == n) return 1;
    int cnt = 0;
    for (int c = 0; c < n; ++c) {
        int a = row - c + n, b = row + c;
        if (col[c] || d1[a] || d2[b]) continue;
        col[c] = d1[a] = d2[b] = 1;
        cnt += solveNQueens(n, row + 1, col, d1, d2);
        col[c] = d1[a] = d2[b] = 0;                // un-choose
    }
    return cnt;
}
```

Family: Palindrome Partitioning, Word Search, Letter Combinations, Restore IP Addresses, Sudoku
Solver — all the same three moves with a different feasibility test.

---

## 13. Dynamic programming

**Core idea:** *the state is the minimal set of facts about the past that the future actually
needs.* Too much state → TLE or MLE. Too little → wrong answer.

The derivation always follows the same four steps:

1. **Write the brute-force recursion with an explicit signature:** `int f(int i, int j, int k)`.
2. **Name the repeated work** — which argument tuples recur?
3. **Memoise on exactly those arguments.** This alone is a correct, interview-acceptable answer.
4. **Flip to bottom-up** and shrink the state if space matters.

Memoisation in C++ is a `vector<vector<int>> memo(m, vector<int>(n, -1));` plus an
`if (memo[i][j] != -1) return memo[i][j];` guard. Use `-1` as "uncomputed" only when `-1` is not a
legal answer; otherwise carry a separate `seen` array.

### 13a. Linear DP

**House Robber** — *rob houses along a street for the maximum total, but you may never rob two
adjacent houses.* → `dp[i] = max(dp[i-1], dp[i-2] + a[i])`.

```
Brute:   recurse take/skip at every index              O(2^n)
DP:      dp array over i                               O(n) time, O(n) space
Optimal: only two cells are ever read -> 2 variables   O(n) time, O(1) space
```

```cpp
int rob(const vector<int>& a) {
    int prev = 0, cur = 0;
    for (int x : a) {
        int nxt = max(cur, prev + x);
        prev = cur;
        cur = nxt;
    }
    return cur;
}
```

**Kadane (maximum subarray)** — *largest sum of any non-empty contiguous subarray; values may be
negative.* The decision at each index is *extend or restart*.

```
Brute:   all (i,j), sum each                           O(n^3) / O(n^2)
Optimal: best subarray ENDING at i, carried forward    O(n)
```

**Unlock:** you cannot carry "the best subarray so far" — it does not compose. You *can* carry "the
best subarray ending exactly here", which does.

```cpp
long long kadane(const vector<int>& a) {
    long long cur = a[0], best = a[0];
    for (int i = 1; i < (int)a.size(); ++i) {
        cur = max((long long)a[i], cur + a[i]);    // restart, or extend
        best = max(best, cur);
    }
    return best;
}
```

Circular variant: `max(kadane, total - minKadane)`, with an all-negative special case.

**Longest Increasing Subsequence** — *length of the longest strictly increasing subsequence.
Subsequence, not subarray: elements need not be adjacent, but order must be preserved.*

```
Brute:   enumerate all subsequences                    O(2^n)
DP:      dp[i] = 1 + max(dp[j]) for j<i, a[j]<a[i]     O(n^2)
Optimal: tails[] + binary search (patience sorting)    O(n log n)
```

**Unlock:** among all increasing subsequences of a given length, only the one with the **smallest
tail** can matter later — so you need one number per length, not one per index.

```cpp
int lis(const vector<int>& a) {
    vector<int> tails;                       // tails[k] = smallest tail of an
    for (int x : a) {                        // increasing subsequence of length k+1
        auto it = lower_bound(tails.begin(), tails.end(), x);  // upper_bound for the
        if (it == tails.end()) tails.push_back(x);             // non-decreasing variant
        else                   *it = x;
    }
    return tails.size();                     // tails is NOT the actual subsequence
}
```

### 13b. Knapsack family

**Problem.** *0/1 knapsack* — items with weight and value, capacity `C`, maximise value taking each
item at most once. *Unbounded* — the same, but each item may be taken any number of times.
*Coin Change* — fewest coins summing to `amount`; *Coin Change II* — how many distinct ways.

**Unlock:** the loop *direction* is the entire difference between the two variants. Descending
guarantees `dp[c - w]` still refers to the previous item's row (each item used once); ascending lets
it refer to the current row (reuse allowed).

```cpp
// 0/1 knapsack — each item once -> iterate capacity DESCENDING
for (auto& it : items)
    for (int c = C; c >= it.w; --c)
        dp[c] = max(dp[c], dp[c - it.w] + it.v);

// Unbounded knapsack — reuse allowed -> iterate capacity ASCENDING
for (auto& it : items)
    for (int c = it.w; c <= C; ++c)
        dp[c] = max(dp[c], dp[c - it.w] + it.v);
```

```cpp
int coinChange(const vector<int>& coins, int amount) {     // fewest coins
    const int INF = INT_MAX / 2;                           // /2 so +1 cannot overflow
    vector<int> dp(amount + 1, INF);
    dp[0] = 0;
    for (int c : coins)
        for (int x = c; x <= amount; ++x)                  // unbounded -> ascending
            dp[x] = min(dp[x], dp[x - c] + 1);
    return dp[amount] >= INF ? -1 : dp[amount];
}

long long coinChangeWays(const vector<int>& coins, int amount) {
    vector<long long> dp(amount + 1, 0);
    dp[0] = 1;
    for (int c : coins)                                    // items OUTER
        for (int x = c; x <= amount; ++x)
            dp[x] += dp[x - c];
    return dp[amount];
}
```

**The loop-order trap:** in `coinChangeWays`, items outer / capacity inner counts **combinations**.
Swap them and you count **permutations** (`1+2` and `2+1` as distinct). Both are legitimate answers
to different questions — know which one you were asked.

Also here: Partition Equal Subset Sum (subset-sum to `total / 2`), Target Sum (reduces to
subset-sum).

### 13c. Two-sequence and grid DP — the `(i, j)` table

```
Brute:   recurse on both suffixes                      O(3^(m+n)) for edit distance
Memo:    cache on (i, j)                               O(mn)
Optimal: bottom-up table, rows rolled                  O(mn) time, O(min(m,n)) space
```

**Problem.** *Edit Distance* — fewest single-character inserts, deletes or replaces to turn `a` into
`b`. *Longest Common Subsequence* — length of the longest sequence appearing in both, in order but
not necessarily contiguously.

**Unlock:** the state is "how much of each string have I consumed" — two indices, nothing more. The
transition is the three edit operations.

```cpp
int editDistance(const string& a, const string& b) {
    int m = a.size(), n = b.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
    for (int i = 0; i <= m; ++i) dp[i][0] = i;             // delete everything
    for (int j = 0; j <= n; ++j) dp[0][j] = j;             // insert everything

    for (int i = 1; i <= m; ++i)
        for (int j = 1; j <= n; ++j)
            if (a[i-1] == b[j-1])
                dp[i][j] = dp[i-1][j-1];
            else
                dp[i][j] = 1 + min({dp[i-1][j],      // delete
                                    dp[i][j-1],      // insert
                                    dp[i-1][j-1]});  // replace
    return dp[m][n];
}

int lcs(const string& a, const string& b) {                // same table, different rule
    int m = a.size(), n = b.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
    for (int i = 1; i <= m; ++i)
        for (int j = 1; j <= n; ++j)
            dp[i][j] = (a[i-1] == b[j-1]) ? dp[i-1][j-1] + 1
                                          : max(dp[i-1][j], dp[i][j-1]);
    return dp[m][n];
}
```

`min({a, b, c})` with braces is the initializer-list overload — the three-argument form does not
exist. The same table shape covers Distinct Subsequences, Regex and Wildcard Matching, Interleaving
String, Unique Paths, Minimum Path Sum. Each row depends only on the previous → two rolling
`vector<int>`s give **O(min(m, n)) space** if asked.

### 13d. Interval DP — `n ≤ 500` and "the last thing you do"

**Problem.** *Burst Balloons* — bursting balloon `i` earns `v[i-1] · v[i] · v[i+1]` using the
neighbours it has **at that moment**; the array then closes up. Maximise total coins.

```
Brute:   try every order of operations                 O(n!)
Optimal: dp[i][j] over ranges, split on the last       O(n^3)
         operation performed inside the range
```

**Unlock for Burst Balloons:** reason about the **last** balloon burst in a range, not the first.
Under "first", the two sides are still coupled through the removed balloon; under "last", both
neighbours are the range's fixed boundaries, so the sides become independent. *Reversing the
decision order is the entire trick.*

```cpp
int burstBalloons(const vector<int>& nums) {
    int n = nums.size();
    vector<int> v(n + 2, 1);                       // pad with virtual 1s
    for (int i = 0; i < n; ++i) v[i + 1] = nums[i];
    vector<vector<int>> dp(n + 2, vector<int>(n + 2, 0));

    for (int len = 1; len <= n; ++len)
        for (int i = 1; i + len - 1 <= n; ++i) {
            int j = i + len - 1;
            for (int k = i; k <= j; ++k)           // k = the LAST balloon burst in [i,j]
                dp[i][j] = max(dp[i][j],
                               dp[i][k-1] + v[i-1] * v[k] * v[j+1] + dp[k+1][j]);
        }
    return dp[1][n];
}
```

Same shape: Matrix Chain Multiplication, Minimum Cost to Cut a Stick, Longest Palindromic
Subsequence.

### 13e. Bitmask DP — `n ≤ 20`

**Problem.** *Travelling Salesman* — visit every node exactly once and return to the start, at
minimum total cost.

```
Brute:   all n! orderings                              O(n!)
Optimal: dp[mask][last] — the SET visited matters,     O(2^n · n^2)
         the order you visited it in does not
```

**Unlock:** `n ≤ 20` in the constraints *is* the instruction to make the subset your state.

```cpp
int tsp(const vector<vector<int>>& d) {
    int n = d.size(), FULL = 1 << n;
    const int INF = INT_MAX / 2;
    vector<vector<int>> dp(FULL, vector<int>(n, INF));
    dp[1][0] = 0;                                  // started at node 0, only it visited

    for (int mask = 1; mask < FULL; ++mask)
        for (int u = 0; u < n; ++u) {
            if (!(mask >> u & 1) || dp[mask][u] >= INF) continue;
            for (int v = 0; v < n; ++v) {
                if (mask >> v & 1) continue;
                int nm = mask | 1 << v;
                dp[nm][v] = min(dp[nm][v], dp[mask][u] + d[u][v]);
            }
        }

    int best = INF;
    for (int u = 0; u < n; ++u) best = min(best, dp[FULL-1][u] + d[u][0]);
    return best;
}
```

Also: Assignment Problem, Partition to K Equal Sum Subsets, Shortest Path Visiting All Nodes.
`1 << n` is an `int` — use `1LL << n` if `n ≥ 31`.

### 13f. State-machine DP — the stock problems

**Problem.** *Best Time to Buy and Sell Stock with Cooldown* — unlimited transactions, one share
at a time, and you must sit out one full day after every sale.

**Unlock:** name the states you can be in, draw the arrows, and the whole
Best-Time-to-Buy-and-Sell family collapses to four lines.

```
Brute:   try every buy/sell pair set                   O(2^n)
Optimal: 2-3 rolling scalars (hold / free / cooldown)  O(n) time, O(1) space
```

```cpp
int maxProfitCooldown(const vector<int>& p) {
    int hold = INT_MIN / 2, free_ = 0, cool = 0;   // /2 so free_ - x cannot overflow
    for (int x : p) {
        int nh = max(hold, free_ - x);             // holding: keep, or buy today
        int nf = max(free_, cool);                 // free: stay free, or leave cooldown
        int nc = hold + x;                         // cooldown: sold today
        hold = nh; free_ = nf; cool = nc;          // all three update SIMULTANEOUSLY
    }
    return max(free_, cool);
}
```

Python's tuple assignment updates all three at once for free; in C++ you must stage them in
temporaries or the second line reads the already-updated first. Write the `nh/nf/nc` temporaries
every time.

---

## 14. Tries

**Core idea:** share prefixes across a dictionary so a query costs O(L) regardless of how many words
you stored. A hash set matches whole strings only; a trie is the structure for *prefix* questions.

**Problem.** *Implement Trie* — support `insert(word)`, `search(word)` (exact) and
`startsWith(prefix)` over a dictionary of `N` words of length up to `L`.

```
Brute:   scan all N words per query                    O(N · L) per query
Better:  hash set — exact match only, no prefixes      O(L), but cannot do prefixes
Optimal: trie                                          O(L) per query, prefixes free
```

```cpp
struct Trie {
    Trie* kids[26] = {};
    bool end = false;

    void insert(const string& w) {
        Trie* node = this;
        for (char c : w) {
            int i = c - 'a';
            if (!node->kids[i]) node->kids[i] = new Trie();
            node = node->kids[i];
        }
        node->end = true;
    }

    bool find(const string& w, bool prefix = false) const {
        const Trie* node = this;
        for (char c : w) {
            int i = c - 'a';
            if (!node->kids[i]) return false;
            node = node->kids[i];
        }
        return prefix ? true : node->end;
    }
};
```

The fixed `Trie* kids[26]` array is faster than a map and fine for lowercase-only input; swap to
`unordered_map<char, Trie*>` if the alphabet is unbounded or memory matters.

**Word Search II** is the payoff case: trie of the dictionary + DFS over the grid, pruning the
instant the current path leaves the trie — that inverts an O(words · cells) search into one grid
walk. Also: maximum XOR pair, via a binary trie over 32 bits.

---

## 15. Bit manipulation

**Core idea:** a bounded universe (≤ 32 or 64 items, or 26 letters) fits in one integer, which turns
set operations into single instructions and a subset-enumeration into an arithmetic loop.

| Trick | Expression |
|---|---|
| Test bit i | `x >> i & 1` |
| Set / clear / toggle bit i | `x \| 1 << i` · `x & ~(1 << i)` · `x ^ 1 << i` |
| Lowest set bit | `x & -x` |
| Clear the lowest set bit | `x & (x - 1)` |
| Popcount | `__builtin_popcount(x)` / `__builtin_popcountll(x)` |
| Power of two | `x > 0 && (x & (x - 1)) == 0` |
| Highest set bit index | `31 - __builtin_clz(x)` (undefined for `x == 0`) |
| Iterate every subset of `mask` | `for (int s = mask; s; s = (s - 1) & mask)` |

### 15a. Single Number

**Problem.** Every value appears exactly twice except one. Find it in O(1) space.

```
Brute:   count occurrences in a hash map               O(n) time, O(n) space
Optimal: XOR everything                                O(n) time, O(1) space
```

**Unlock:** `x ^ x == 0` and XOR is commutative, so every pair annihilates regardless of position —
you never needed to know *which* elements paired up.

```cpp
int singleNumber(const vector<int>& a) {
    int x = 0;
    for (int v : a) x ^= v;
    return x;
}
```

### 15b. Single Number II — every other element appears three times

**Problem.** Every value appears exactly three times except one. XOR no longer cancels.

**Unlock:** XOR only cancels in pairs. Generalise it: count each bit position independently and take
the count mod 3. The bounded universe here is *32 bit positions*, not the array.

```cpp
int singleNumberII(const vector<int>& a) {
    unsigned res = 0;
    for (int b = 0; b < 32; ++b) {
        int cnt = 0;
        for (int v : a) cnt += ((unsigned)v >> b) & 1u;
        if (cnt % 3) res |= 1u << b;               // unsigned: bit 31 is not UB
    }
    return (int)res;
}
```

Missing Number → XOR all indices with all values, same annihilation argument.

**C++ traps:** `1 << 31` overflows a signed `int` — write `1LL << k` for `k ≥ 31`. Shifting by
`≥ 32` on a 32-bit type is undefined behaviour, not zero. Right-shifting a negative `int` is
implementation-defined; use `unsigned` when you mean a logical shift.

---

## 16. Cyclic sort / index-as-hash

**Core idea:** when values live in `[1..n]` or `[0..n-1]`, **the array is your hash table** — value
`v` belongs at index `v - 1`. That is how you hit O(1) extra space.

```
Brute:   sort, then scan                               O(n log n)
Better:  hash set of seen values                       O(n) time, O(n) space
Optimal: swap each value to its home index             O(n) time, O(1) space
```

**Problem.** *First Missing Positive* — smallest positive integer **not** present, in O(n) time and
O(1) extra space. Related: *Find All Duplicates* — with values in `[1..n]` each appearing once or
twice, report the repeated ones.

**Unlock:** the O(1)-space demand in the statement is the hint. Ask whether you may mutate the
input — if yes, the input *is* the auxiliary structure you were told you could not allocate.

```cpp
int firstMissingPositive(vector<int>& a) {
    int n = a.size();
    for (int i = 0; i < n; ++i)
        while (a[i] >= 1 && a[i] <= n && a[a[i] - 1] != a[i])
            swap(a[a[i] - 1], a[i]);               // swap it into place
    for (int i = 0; i < n; ++i)
        if (a[i] != i + 1) return i + 1;
    return n + 1;
}
```

The nested `while` looks quadratic but each swap places one element permanently, so it is O(n).

> **Watch this one line:** `swap(a[a[i] - 1], a[i])` is fine because `swap` binds both references
> before writing. Writing it out as `int t = a[i]; a[i] = a[t - 1]; a[t - 1] = t;` also works — but
> `a[a[i] - 1] = a[i]; a[i] = ...` does **not**, because the first assignment changes `a[i]` and
> corrupts the index the second one reads. Use `std::swap`.

**Non-destructive variant** — mark presence by negating, when values must survive:

```cpp
vector<int> findDuplicates(vector<int>& a) {       // values in [1..n], each 1 or 2 times
    vector<int> res;
    for (int x : a) {
        int i = abs(x) - 1;
        if (a[i] < 0) res.push_back(abs(x));       // already marked -> seen twice
        else          a[i] = -a[i];                // mark index abs(x)-1 as seen
    }
    return res;
}
```

Find All Disappeared Numbers is the same loop, reading off the still-positive indices at the end.
Find the Duplicate Number with a **read-only** array → Floyd cycle detection (§9b).

---

## 17. Matrix odds and ends

**Core idea:** most matrix problems are a coordinate transform plus a place to stash state. Say the
transform in words before you write indices.

### 17a. Rotate 90° in place

**Problem.** *Rotate Image* — rotate an `n × n` matrix 90° clockwise, **in place**.

**Unlock:** rotation = transpose, then reverse each row. Two simple passes beat one confusing
four-way cycle swap.

```cpp
void rotate(vector<vector<int>>& m) {
    int n = m.size();
    for (int i = 0; i < n; ++i)                              // transpose
        for (int j = i + 1; j < n; ++j)
            swap(m[i][j], m[j][i]);
    for (auto& row : m) reverse(row.begin(), row.end());     // then reverse rows
}
```

### 17b. Set Matrix Zeroes in O(1) space

**Problem.** If any cell is 0, zero its entire row and column — using O(1) extra space.

```
Brute:   copy the matrix, write zeros into the copy     O(mn) space
Better:  two boolean arrays for rows and columns        O(m + n) space
Optimal: use row 0 and column 0 AS those arrays         O(1) space
```

**Unlock:** the markers you need are exactly as many as one row plus one column — and you already
own a row and a column. Column 0 needs a separate flag because cell `(0,0)` is shared between the
two markers.

```cpp
void setZeroes(vector<vector<int>>& m) {
    int R = m.size(), C = m[0].size();
    bool col0 = false;
    for (int r = 0; r < R; ++r) {
        if (m[r][0] == 0) col0 = true;
        for (int c = 1; c < C; ++c)
            if (m[r][c] == 0) { m[r][0] = 0; m[0][c] = 0; }  // mark in the margins
    }
    for (int r = R - 1; r >= 0; --r) {                       // bottom-up: row 0 LAST
        for (int c = C - 1; c >= 1; --c)
            if (m[r][0] == 0 || m[0][c] == 0) m[r][c] = 0;
        if (col0) m[r][0] = 0;
    }
}
```

The second loop must run bottom-up, or you overwrite the markers in row 0 before reading them.

### 17c. Search a matrix with sorted rows and columns

**Problem.** *Search a 2D Matrix II* — each row increases left-to-right and each column increases
top-to-bottom, but rows are **not** globally ordered. Is `target` present?

```
Brute:   scan every cell                               O(mn)
Better:  binary search each row                        O(m log n)
Optimal: staircase walk from a corner                  O(m + n)
```

**Unlock:** the top-right corner is the only cell that is simultaneously the max of its row and the
min of its column — so one comparison eliminates a whole row or a whole column.

```cpp
bool searchMatrix(const vector<vector<int>>& m, int target) {
    int r = 0, c = (int)m[0].size() - 1;           // start at the TOP-RIGHT
    while (r < (int)m.size() && c >= 0) {
        if (m[r][c] == target) return true;
        if (m[r][c] > target) --c;                 // whole column is too big
        else                  ++r;                 // whole row is too small
    }
    return false;
}
```

---

## The four things to say in every round

Your tracked weak spots, turned into a checklist. Say each one out loud, unprompted.

1. **Clarify semantics before approach.** Sorted? Duplicates? Negatives? Empty input? May I mutate
   the input? What is the tie-break / output-ordering rule? What is the value range — do I need
   `long long`?
2. **Convert n into a budget immediately:** *"n ≤ 10^5, so I'm aiming for n log n; O(n²) is 10^10
   and dead."* Then say what that budget **forbids** — that is what selects the pattern.
3. **State the brute force as a function signature, then name the repeated work,** before you
   optimise. Never jump silently to the optimal, and never say "I can't optimise this" without first
   checking (a) auxiliary space and (b) which constraint you have not yet spent.
4. **Dry-run one input you invented** — not the given example — before declaring done. Pick an edge
   case: empty, single element, all equal, all negative.

And when you are stuck: **think out loud.** Silence reads as no progress; a wrong idea narrated
reads as progress, and usually earns you a hint.

---

## C++-specific round killers

Things that cost real interview points in this language, none of them algorithmic:

- `priority_queue<int>` is a **max**-heap. Min-heap is
  `priority_queue<int, vector<int>, greater<int>>`.
- `int` overflow in prefix sums, `accumulate(..., 0)`, `(lo + hi) / 2`, and `width * height`.
- Passing `vector`s by value into a recursive helper — exponential copying. Use `&`.
- `unordered_map::operator[]` **inserts** on read. Use `.find()` or `.count()` when you only mean to
  look.
- `unordered_map` with a `pair` key does not compile without a custom hash — encode into a single
  integer or use `map`.
- Erasing from a container while iterating — capture `it = c.erase(it)`.
- `min(a, b, c)` does not exist; write `min({a, b, c})`.
- `std::stack` cannot be iterated — use a `vector` when you need to look below the top.
- Naming a Union-Find method `union` — it is a keyword.
- Simultaneous updates (state-machine DP) need staged temporaries; C++ has no tuple assignment.
