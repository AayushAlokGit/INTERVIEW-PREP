# DSA Pattern Speedrun

Cram sheet: the ~16 patterns that cover the large majority of interview problems, each shown as a
**ladder** — brute force → each optimisation step, with *the observation that unlocks the step*.
The observation is the thing you say out loud in an interview. The code is secondary.

Code is C++ (`using namespace std;` assumed, headers omitted).

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
`sum(nums)` as a binary-search bound, `ht * width` in the histogram problem, and `l + r` inside a
binary search all need `long long` or a subtraction-based midpoint.

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

**Traps:** duplicate skipping goes *after* you record a hit, not before. Two pointers on
**unsorted** data is only valid with an independent monotonic argument (Container With Most Water is
the example).

---

## 2. Sliding window

**Trigger:** contiguous window + a metric that is **monotone** as the window grows. All-positive
values is the classic guarantee. Negatives kill it → go to prefix sums instead.

### Fixed window

```cpp
long long s = 0;
for (int i = 0; i < k; ++i) s += a[i];
long long best = s;
for (int i = k; i < (int)a.size(); ++i) {
    s += a[i] - a[i-k];              // add the new, drop the old
    best = max(best, s);
}
```

**Unlock:** the O(n·k) recompute shares all but two elements between consecutive windows.

### Variable window — longest valid

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

### Variable window — shortest valid (Minimum Window Substring)

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

**The "exactly k" trick:** `exactly(k) = atMost(k) - atMost(k-1)`. A direct "exactly k" window is
not monotone; "at most k" is. Use it for *Subarrays with K Different Integers* and *Binary Subarrays
With Sum*.

**Traps:** the shrink condition is a `while`, never an `if`. Record `best` at the right moment —
longest → after expanding; shortest → inside the shrink loop. Prefer a fixed `vector<int>` over
`unordered_map` when the alphabet is bounded — it is both faster and easier to reason about.

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
Note `unordered_map::operator[]` value-initialises to `0`, so `++seen[p]` needs no guard — but
*reading* with `[]` also inserts, which is why the lookup above uses `find`.

**Variants worth 30 seconds each:**

- Sum divisible by k → key on `p % k`, normalising negatives with `((p % k) + k) % k`.
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

### Ladder — Longest Consecutive Sequence

```
Brute:   sort, then scan runs                          O(n log n)
Optimal: hash set; only walk from x where x-1 absent   O(n)
```

**Unlock:** the "only start at sequence heads" guard is what makes the total work linear — every
element is visited by exactly one chain walk. Without the guard it degrades to O(n²).

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

**Group Anagrams:** key by the sorted string, or by a 26-length count signature for O(n·L).

**Trap:** claiming O(n) while rebuilding a frequency map inside a loop over the same n. That is
O(n²). Also, `unordered_map` with a `pair`/`vector` key needs a custom hash — reach for `map` or
encode the key into a single integer instead of writing one under time pressure.

---

## 5. Binary search

Two distinct uses. Recognising the second is the higher-value skill and the more common miss.

### 5a. Binary search on an index

```cpp
int lo = 0, hi = n;                    // half-open [lo, hi) kills most off-by-ones
while (lo < hi) {
    int mid = lo + (hi - lo) / 2;      // never (lo + hi) / 2 — overflow
    if (ok(mid)) hi = mid;             // mid might be the answer, keep it in range
    else         lo = mid + 1;
}
return lo;                             // first index where ok() is true
```

Use this *one* template for lower bound, upper bound and first-true. Never write a three-way
`== target` search — it does not generalise to duplicates or to boundary queries.

The STL gives you the common cases free: `lower_bound(v.begin(), v.end(), x)` is the first element
`>= x`, `upper_bound` the first `> x`, and `distance`/pointer subtraction converts to an index. Say
you'd use them, then hand-roll if the interviewer wants the template.

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

```cpp
int splitArray(const vector<int>& nums, int k) {     // min possible largest subarray sum
    auto ok = [&](long long cap) {
        int parts = 1;
        long long cur = 0;
        for (int x : nums) {
            if (cur + x > cap) { ++parts; cur = x; }
            else                 cur += x;
        }
        return parts <= k;
    };

    long long lo = *max_element(nums.begin(), nums.end());
    long long hi = accumulate(nums.begin(), nums.end(), 0LL);   // 0LL, not 0
    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (ok(mid)) hi = mid;
        else         lo = mid + 1;
    }
    return (int)lo;
}
```

The exact same shape solves Koko Eating Bananas, Capacity to Ship Packages in D Days, Minimum Days
to Make M Bouquets, Kth Smallest Element in a Sorted Matrix, and Kth Smallest Pair Distance.

**Traps:** `lo` and `hi` must bracket a feasible range — derive both bounds out loud. Prove
monotonicity of `ok()`. If you ever write `lo = mid`, you must use `mid = lo + (hi - lo + 1) / 2` or
you infinite-loop. `accumulate(..., 0)` deduces `int` and overflows — pass `0LL`.

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

```cpp
vector<vector<int>> merge(vector<vector<int>>& iv) {
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

A custom key is a lambda comparator — for "by end ascending":
`sort(iv.begin(), iv.end(), [](auto& a, auto& b){ return a[1] < b[1]; });`

### Sweep line — Minimum Meeting Rooms

```
Brute:   for each meeting, count overlaps              O(n^2)
Optimal: sort starts and ends separately, +1/-1 sweep  O(n log n)
```

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

### Greedy + regret heap — the one that catches people

When you only learn a choice was wrong *after* you have passed it: **take everything, then undo the
worst commitment.**

```cpp
int scheduleCourse(vector<vector<int>>& c) {       // Course Schedule III
    sort(c.begin(), c.end(),
         [](auto& a, auto& b){ return a[1] < b[1]; });   // by deadline
    priority_queue<int> taken;                     // max-heap of durations
    long long total = 0;
    for (auto& x : c) {
        taken.push(x[0]);
        total += x[0];
        if (total > x[1]) {                        // over budget -> undo the worst
            total -= taken.top();
            taken.pop();
        }
    }
    return taken.size();
}
```

Same shape: *Furthest Building* (use a ladder on every climb; when out, demote the smallest
ladder-use to bricks — a **min**-heap there), *IPO / Maximum Events* (heap of currently-available
options, take the best each step).

**Diagnostic when a greedy breaks:** *is my candidate set too small, or is my move set too small?*
Move set too small → add an undo (a heap or a stack). Candidate set too small → widen what you
consider.

**Fails when** retraction is not legal, because an earlier commitment irreversibly changed the
world. Then it is DP, not greedy.

**C++ note:** `priority_queue<int>` is a **max**-heap. For a min-heap write
`priority_queue<int, vector<int>, greater<int>>`. Getting this backwards is a classic round-killer —
say which one you want before you type it.

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

```cpp
vector<int> nextGreater(const vector<int>& a) {
    int n = a.size();
    vector<int> res(n, -1);
    stack<int> st;                                 // indices, values decreasing
    for (int i = 0; i < n; ++i) {
        while (!st.empty() && a[st.top()] < a[i]) {
            res[st.top()] = a[i];                  // a[i] answers everything it beats
            st.pop();
        }
        st.push(i);
    }
    return res;
}
```

### Ladder — Largest Rectangle in Histogram

```
Brute:   for each bar, expand both directions          O(n^2)
Optimal: increasing stack; when a bar pops, its width  O(n)
         is bounded left by the new stack top and
         right by the current index
```

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

### Ladder — Sliding Window Maximum (monotonic deque)

```
Brute:   max() per window                              O(n·k)
Better:  max-heap with lazy deletion                   O(n log n)
Optimal: deque of decreasing indices                   O(n)
```

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

**Unlock:** an element that is both older *and* smaller can never be an answer again, so drop it
permanently.

**Also stack-shaped:** Trapping Rain Water (stack, or two pointers, or prefix-max arrays — know two
of the three), Daily Temperatures, Remove K Digits, Remove Duplicate Letters, Basic Calculator,
Asteroid Collision, Valid Parentheses, Decode String.

**C++ note:** `std::stack` has no iteration. If you need to inspect below the top, use a
`vector<int>` as your stack (`back()` / `pop_back()`) — that is the common competitive idiom and it
costs you nothing.

---

## 8. Heaps and top-K

**Trigger:** "k largest", "k closest", "median of a stream", "merge k sorted", or repeatedly taking
the current best.

### Ladder — Kth Largest Element

```
Brute:   sort, index                                   O(n log n)
Optimal: min-heap of size k                            O(n log k)
Better:  nth_element (quickselect)                     O(n) average
Stream:  min-heap of size k is the only option         O(log k) per element
```

Rule to memorise: **kth largest → min-heap of size k** (you pop the smallest). Kth smallest →
max-heap of size k.

```cpp
int kthLargest(const vector<int>& a, int k) {
    priority_queue<int, vector<int>, greater<int>> pq;     // MIN-heap
    for (int x : a) {
        pq.push(x);
        if ((int)pq.size() > k) pq.pop();                  // evict the smallest
    }
    return pq.top();
}

// One-shot, mutable input -> quickselect via the STL:
int kthLargestFast(vector<int> a, int k) {
    nth_element(a.begin(), a.begin() + (k - 1), a.end(), greater<int>());
    return a[k - 1];
}
```

### Merge K Sorted Lists

```
Brute:   concatenate and sort                          O(N log N)
Optimal: heap of the k current heads                   O(N log k)
Alt:     pairwise merge, log k rounds                  O(N log k)
```

```cpp
ListNode* mergeKLists(vector<ListNode*>& lists) {
    auto cmp = [](ListNode* a, ListNode* b){ return a->val > b->val; };  // min-heap
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

Note the comparator is **inverted**: `priority_queue` puts the *largest* under `top()` by the
comparator's ordering, so `a->val > b->val` yields a min-heap.

### Two heaps — median of a data stream

Max-heap over the lower half, min-heap over the upper half, sizes differing by at most 1, rebalance
after every insert. The same structure powers Sliding Window Median (plus lazy deletion, or a
`multiset` with a mid-iterator).

---

## 9. Linked lists and fast/slow pointers

Three primitives compose into nearly every list problem:

```cpp
ListNode* reverse(ListNode* head) {
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

**Cycle entry point:** after the meeting, reset one pointer to `head` and advance both one step at a
time — they meet at the entry. The same math solves *Find the Duplicate Number* (treat the array as
a functional graph) in O(1) space with a read-only array.

**Always use a dummy head** when the head itself may be removed — `ListNode dummy(0); dummy.next =
head;` then return `dummy.next`. Reorder List = middle + reverse + merge. Palindrome Linked List =
the same, in O(1) space.

---

## 10. Trees

Almost everything reduces to: **recurse, and decide what the child returns to the parent.** That
return value *is* the design decision.

```cpp
int depth(TreeNode* root) {                        // returns a scalar upward
    if (!root) return 0;
    return 1 + max(depth(root->left), depth(root->right));
}
```

### Ladder — Diameter of a Binary Tree

```
Brute:   at each node, depth(left) + depth(right)      O(n^2)
Optimal: one post-order pass — return the depth the    O(n)
         parent needs, update a global best en route
```

```cpp
int best = 0;
int dfs(TreeNode* node) {
    if (!node) return 0;
    int l = dfs(node->left), r = dfs(node->right);
    best = max(best, l + r);        // the answer THROUGH this node
    return 1 + max(l, r);           // what the PARENT needs
}
```

**Unlock:** the answer *through* a node and the value *needed by* the parent are two different
things. Return the parent's need; accumulate the answer in a side variable. This single move solves
Diameter, Binary Tree Maximum Path Sum, House Robber III, Balanced Binary Tree and Longest Univalue
Path.

Prefer passing the accumulator by reference (`int& best`) or wrapping it in a member over a true
global — say that out loud, then write whichever is faster.

### Must-know facts

- **BST inorder traversal is sorted** → validate BST, kth smallest, inorder successor, recover BST.
- Validating a BST needs `(low, high)` bounds passed **down** — and they must be `long long` (or
  `LONG_MIN`/`LONG_MAX` sentinels) because node values can be `INT_MIN`/`INT_MAX`. A local parent
  comparison is wrong regardless.
- **LCA:** if root is p or q → root; else recurse both sides; two non-null results → root; otherwise
  the non-null one.
- **BFS by level:** snapshot `int sz = q.size();` at the top of each round, then loop `sz` times.
- Serialize/deserialize: preorder with explicit `#` markers for nulls.
- Build from preorder + inorder: precompute an `unordered_map<int,int>` of inorder positions → O(n)
  not O(n²).

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

```cpp
int bfs(vector<vector<char>>& g, vector<pair<int,int>> starts) {
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

The `dr[]`/`dc[]` direction arrays are worth writing from muscle memory — they remove four
copy-pasted neighbour lines and the bugs that hide in them.

**The off-by-one to watch:** a bare `++d` at the bottom of the outer loop counts *levels*, so it
returns `maxDistance + 1`. The `if (!q.empty())` guard is what makes `d` a distance. Rotting Oranges
wants the distance; "number of levels" is almost never the thing asked for. Say which one you are
returning.

**Multi-source BFS** (Rotting Oranges, 01 Matrix, Walls and Gates): push *every* source at distance
0 before you start. That turns an O(sources · V) problem into O(V) — a very common intended
optimisation.

### Topological sort (Kahn)

```cpp
vector<int> topo(int n, vector<pair<int,int>>& edges) {
    vector<vector<int>> g(n);
    vector<int> indeg(n, 0);
    for (auto& [u, v] : edges) { g[u].push_back(v); ++indeg[v]; }

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

Cycle detection in a **directed** graph via DFS needs three colours (`0` unvisited / `1` on the
current stack / `2` done). A plain visited array is only correct for undirected graphs.

### Union-Find

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
        if (a == b) return false;
        if (r[a] < r[b]) swap(a, b);
        p[b] = a;
        if (r[a] == r[b]) ++r[a];
        return true;
    }
};
```

Use it when merges arrive **online**, or for Kruskal, Accounts Merge, Redundant Connection, Number
of Islands II. Do not name the method `union` — it is a C++ keyword and will not compile.

### Dijkstra

```cpp
vector<long long> dijkstra(const vector<vector<pair<int,int>>>& g, int src, int n) {
    const long long INF = LLONG_MAX / 4;
    vector<long long> dist(n, INF);
    dist[src] = 0;

    priority_queue<pair<long long,int>,
                   vector<pair<long long,int>>,
                   greater<>> pq;                 // MIN-heap on distance
    pq.push({0, src});

    while (!pq.empty()) {
        auto [d, u] = pq.top(); pq.pop();
        if (d > dist[u]) continue;                // stale entry, lazy deletion
        for (auto& [v, w] : g[u]) {
            if (d + w < dist[v]) {
                dist[v] = d + w;
                pq.push({dist[v], v});
            }
        }
    }
    return dist;
}
```

The `if (d > dist[u]) continue;` line is not optional — it is what keeps this O(E log V). Use
`LLONG_MAX / 4` rather than `LLONG_MAX` so `d + w` cannot overflow.

---

## 12. Backtracking

**Trigger:** enumerate *all* valid configurations. The template is always three moves:
choose → recurse → un-choose.

```cpp
void bt(int i, vector<int>& nums, vector<int>& path, vector<vector<int>>& res) {
    if (i == (int)nums.size()) { res.push_back(path); return; }   // copies path
    bt(i + 1, nums, path, res);                                   // skip nums[i]
    path.push_back(nums[i]);
    bt(i + 1, nums, path, res);
    path.pop_back();                                              // un-choose
}
```

`res.push_back(path)` copies the vector, which is what you want — `path` keeps mutating. Pass
`path` and `res` by **reference**; passing by value silently turns this into an exponential-copy
disaster and is a common C++-specific bug.

### Combination Sum (unbounded reuse)

```cpp
void bt(int start, int rem, vector<int>& c, vector<int>& path,
        vector<vector<int>>& res) {
    if (rem == 0) { res.push_back(path); return; }
    for (int i = start; i < (int)c.size(); ++i) {
        if (c[i] > rem) break;                     // prune — needs the sort
        path.push_back(c[i]);
        bt(i, rem - c[i], c, path, res);           // i, not i+1 -> reuse allowed
        path.pop_back();
    }
}

vector<vector<int>> combinationSum(vector<int>& c, int target) {
    sort(c.begin(), c.end());
    vector<int> path;
    vector<vector<int>> res;
    bt(0, target, c, path, res);
    return res;
}
```

**Duplicates in the input:** sort, then inside the loop
`if (i > start && c[i] == c[i-1]) continue;` — this skips duplicate *branches* at the same depth
while still allowing the same value to appear deeper.

**Complexity** is (number of solutions) × (cost to build one). Say it that way rather than
"exponential".

Family: Permutations, Subsets I/II, Combination Sum I/II, Palindrome Partitioning, Word Search,
N-Queens, Letter Combinations, Restore IP Addresses, Sudoku Solver.

**Pruning is the whole game:** sort plus an early `break`, feasibility bounds, and for N-Queens track
occupied columns and diagonals in three `vector<bool>`s keyed on `col`, `r - c + n` and `r + c`
instead of rescanning the board.

---

## 13. Dynamic programming

The most reliable derivation route on the sheet, and it always follows the same four steps:

1. **Write the brute-force recursion with an explicit signature:** `int f(int i, int j, int k)`.
2. **Name the repeated work** — which argument tuples recur?
3. **Memoise on exactly those arguments.** This alone is a correct, interview-acceptable answer.
4. **Flip to bottom-up** and shrink the state if space matters.

*The state is the minimal set of facts about the past that the future actually needs.* Too much
state → TLE or MLE. Too little → wrong answer.

Memoisation in C++ is a `vector<vector<int>> memo(m, vector<int>(n, -1));` plus an
`if (memo[i][j] != -1) return memo[i][j];` guard. Use `-1` as "uncomputed" only when `-1` is not a
legal answer; otherwise carry a separate `seen` array.

### 13a. Linear DP

**House Robber** — `dp[i] = max(dp[i-1], dp[i-2] + a[i])`, collapsed to two rolling variables:

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

**Longest Increasing Subsequence**

```
Brute:   enumerate all subsequences                    O(2^n)
DP:      dp[i] = 1 + max(dp[j]) for j<i, a[j]<a[i]     O(n^2)
Optimal: tails[] + binary search (patience sorting)    O(n log n)
```

```cpp
int lis(const vector<int>& a) {
    vector<int> tails;
    for (int x : a) {
        auto it = lower_bound(tails.begin(), tails.end(), x);  // upper_bound for
        if (it == tails.end()) tails.push_back(x);             // the non-decreasing
        else                   *it = x;                        // variant
    }
    return tails.size();                     // tails is NOT the actual subsequence
}
```

**Kadane (maximum subarray)** — `cur = max(x, cur + x)`; the decision at each step is "extend or
restart". The circular variant is `max(kadane, total - minKadane)`, with an all-negative special
case.

### 13b. Knapsack family

```cpp
// 0/1 knapsack — each item once -> iterate capacity DESCENDING
for (auto& [w, v] : items)
    for (int c = C; c >= w; --c)
        dp[c] = max(dp[c], dp[c - w] + v);

// Unbounded knapsack — reuse allowed -> iterate capacity ASCENDING
for (auto& [w, v] : items)
    for (int c = w; c <= C; ++c)
        dp[c] = max(dp[c], dp[c - w] + v);
```

The loop *direction* is the entire difference. Descending guarantees `dp[c - w]` still refers to the
previous item's row.

Also here: Coin Change (min coins → `min`, initialise to a large sentinel like `INT_MAX / 2` so
`+1` cannot overflow), Coin Change II (count ways → **items in the outer loop, capacity inner**, or
you count permutations instead of combinations), Partition Equal Subset Sum (subset-sum to
`total / 2`), Target Sum (reduces to subset-sum).

### 13c. Two-sequence and grid DP — the `(i, j)` table

**Edit Distance**

```cpp
int editDistance(const string& a, const string& b) {
    int m = a.size(), n = b.size();
    vector<vector<int>> dp(m + 1, vector<int>(n + 1, 0));
    for (int i = 0; i <= m; ++i) dp[i][0] = i;
    for (int j = 0; j <= n; ++j) dp[0][j] = j;

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
```

`min({a, b, c})` with braces is the initializer-list overload — the three-argument form does not
exist. The same table shape covers LCS, Distinct Subsequences, Regex and Wildcard Matching,
Interleaving String, Unique Paths, Minimum Path Sum. Each row depends only on the previous one →
**O(min(m, n)) space** if asked (two rolling `vector<int>`s).

### 13d. Interval DP — `n ≤ 500` and "the last thing you do"

Burst Balloons, Matrix Chain Multiplication, Minimum Cost to Cut a Stick, Longest Palindromic
Subsequence.

```cpp
for (int len = 2; len <= n; ++len)
    for (int i = 0; i + len - 1 < n; ++i) {
        int j = i + len - 1;
        for (int k = i; k < j; ++k)
            dp[i][j] = best(dp[i][j], dp[i][k] + dp[k+1][j] + cost(i, k, j));
    }
```

**Unlock for Burst Balloons:** reason about the **last** balloon burst in a range, not the first —
only then do the two sides become independent. Reversing the decision order is the entire trick.

### 13e. Bitmask DP — `n ≤ 20`

`dp[mask]` = the best result over the set of used items. Travelling Salesman, Assignment Problem,
Partition to K Equal Sum Subsets, Shortest Path Visiting All Nodes. Costs O(2^n · n) or O(2^n · n²).
Size the array `vector<int> dp(1 << n, ...)` and remember `1 << n` is `int` — use `1LL << n` if
`n ≥ 31`.

### 13f. State-machine DP — the stock problems

Track `hold` / `free` (plus `cooldown`, or times k transactions). Draw the transitions as a small
state machine and the whole Best-Time-to-Buy-and-Sell family collapses to four lines.

```cpp
int maxProfitCooldown(const vector<int>& p) {
    int hold = INT_MIN / 2, free_ = 0, cool = 0;   // /2 so `free_ - x` cannot overflow
    for (int x : p) {
        int nh = max(hold, free_ - x);
        int nf = max(free_, cool);
        int nc = hold + x;
        hold = nh; free_ = nf; cool = nc;          // all three update simultaneously
    }
    return max(free_, cool);
}
```

Python's tuple assignment updates all three at once for free; in C++ you must stage them in
temporaries or the second line reads the already-updated first. This is a real bug source — write
the `nh/nf/nc` temporaries every time.

---

## 14. Tries

**Trigger:** prefix queries, autocomplete, word dictionaries, or searching many words inside one
text.

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

    bool find(const string& w, bool prefix = false) {
        Trie* node = this;
        for (char c : w) {
            int i = c - 'a';
            if (!node->kids[i]) return false;
            node = node->kids[i];
        }
        return prefix ? true : node->end;
    }
};
```

The fixed `Trie* kids[26]` array is faster than a map and fine for lowercase-only inputs; swap to
`unordered_map<char, Trie*>` if the alphabet is unbounded or memory matters. Insert and search are
O(L), independent of dictionary size — that is the win over a hash set, which cannot answer prefix
queries at all. **Word Search II** = trie of the dictionary + DFS over the grid, pruning the instant
the current path leaves the trie. Also: maximum XOR pair, via a binary trie over 32 bits.

---

## 15. Bit manipulation

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

**XOR identities:** `x ^ x = 0`, `x ^ 0 = x`, and it is commutative. Single Number becomes a
one-line O(1)-space fold. Single Number II (every element appears three times) → count bits mod 3.
Missing Number → XOR all indices with all values.

**C++ traps:** `1 << 31` overflows a signed `int` — write `1LL << k` for `k ≥ 31`. Shifting by
`≥ 32` on a 32-bit type is undefined behaviour, not zero. Right-shifting a negative `int` is
implementation-defined; use `unsigned` when you mean a logical shift.

---

## 16. Cyclic sort / index-as-hash

**Trigger:** values live in `[1..n]` or `[0..n-1]` **and** O(1) extra space is demanded.

**Unlock:** the array *is* your hash table — value `v` belongs at index `v - 1`.

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
> before writing. Writing it out manually as
> `int t = a[i]; a[i] = a[t - 1]; a[t - 1] = t;` also works — but
> `a[a[i] - 1] = a[i]; a[i] = ...` does **not**, because the first assignment changes `a[i]` and
> corrupts the index the second one reads. Use `std::swap`.

Non-destructive variant: mark presence by negating `a[abs(x) - 1]` — that solves Find All Duplicates
and Find All Disappeared Numbers. Find the Duplicate Number with a read-only array → Floyd cycle
detection (§9).

---

## 17. Matrix odds and ends

- **Rotate 90°** = transpose, then reverse each row (`for (auto& row : m) reverse(row.begin(),
  row.end());`). Say it, then write it.
- **Set Matrix Zeroes in O(1) space:** use row 0 and column 0 as the marker arrays, with a separate
  flag for column 0 itself.
- **Search a 2D matrix with sorted rows and columns:** start at the top-right; `>` → move left,
  `<` → move down. O(m + n) staircase walk.

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

A short list of the things that cost real interview points in this language, none of which are
algorithmic:

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
