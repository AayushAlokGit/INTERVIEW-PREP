# DSA Round Transcript
**Date:** 2026-07-06
**Start Time:** 10:43
**End Time:** 11:18
**Duration:** 35 minutes
**Problem:** Russian Doll Envelopes
**Topic:** Sorting + Longest Increasing Subsequence (Binary Search)
**Difficulty:** Hard

---

## Problem Statement
You are given a 2D array `envelopes` where `envelopes[i] = [wi, hi]` represents the width and height of an envelope. One envelope can fit into another if and only if **both** its width and height are **strictly greater** than the other envelope's. Return the maximum number of envelopes you can Russian-doll (nested). You cannot rotate an envelope.

**Example 1:**
```
Input:  envelopes = [[5,4],[6,4],[6,7],[2,3]]
Output: 3
Explanation: [2,3] => [5,4] => [6,7].
```
**Example 2:**
```
Input:  envelopes = [[1,1],[1,1],[1,1]]
Output: 1
Explanation: Identical envelopes can't nest (not strictly greater).
```

Constraints:
- `1 <= envelopes.length <= 10^5`
- `envelopes[i].length == 2`
- `1 <= wi, hi <= 10^5`

*Note: this round replaced "Evaluate Division," which was flagged mid-setup as already solved (2026-05-07).*

---

## Conversation Log

**Interviewer:** Presented problem, examples, constraints. Asked for clarifying questions.

**Aayush:** What are the constraints?

**Interviewer:** Restated: up to 100,000 envelopes, always [width,height], values 1..10^5, duplicates allowed, nesting requires strictly greater in both dimensions.

**Aayush:** Both height and width vary, so fix the order across one dimension by sorting by width ascending. Then to Russian-doll we need height strictly ascending irrespective of position — so it's LIS with respect to height. LIS in n log n using tails.

**Interviewer:** Right skeleton. Pressure-tested with a tie: `[[3,3],[3,4]]` — both width 3. Sorted ascending → `[[3,3],[3,4]]`, LIS on heights [3,4] returns 2, but they can't nest. What does that tell you about equal widths?

**Aayush:** For the same width, sort the heights in descending order so that when finding LIS there is no possibility that for the same width a lower height can come before.

**Interviewer:** Exactly — width ascending, height descending on ties, so a strictly increasing subsequence picks at most one equal-width envelope. Asked him to code it and be precise about how the LIS enforces strictly increasing.

**Aayush:** Wrote the C++ solution (below): custom comparator (width asc, height desc on ties), then LIS via `tails` using `lower_bound`.

**Interviewer:** Noted the leftover `dfs`/`dirs` block pasted from the previous problem is dead code. Then asked: (1) why `lower_bound` not `upper_bound`, what breaks with `upper_bound`; (2) dry-run the height stream for `[[3,3],[3,4]]` through the tails loop.

**Aayush:** (1) lower_bound gives the position for `>=` (finds the sequence ending at this element), whereas upper_bound gives `>`. (2) After sorting -> {{3,4},{3,3}}: i=0 tails -> {4}; i=1 tails -> {3}.

**Interviewer:** Both correct. Closed the loop: with upper_bound you'd get the longest non-decreasing subsequence, letting equal heights extend and breaking strictness. Dry-run lands on tails {3}, answer 1 — correct. Asked for time and space complexity including the sort.

**Aayush:** n log n sort, and n for the tails array.

**Interviewer:** Pushed — the tails loop does a lower_bound each of n iterations; what does that make the loop?

**Aayush:** n log n — sort + n log n — tails loop, and n for tails array.

**Interviewer:** Correct and complete — optimal solution. Moved to feedback.

---

## Solution
**Aayush's Final Solution:**
```cpp
int maxEnvelopes(vector<vector<int>>& envelopes) {
    int n = envelopes.size();
    // asc by width, descending height for same width
    sort(envelopes.begin(), envelopes.end(),
         [](const vector<int>& a, const vector<int>& b){
             if(a[0] == b[0]) return a[1] > b[1];
             return a[0] < b[0];
         });

    vector<int> tails;
    for(int i = 0; i < n; i++) {
        int idx = lower_bound(tails.begin(), tails.end(), envelopes[i][1]) - tails.begin();
        if(idx == (int)tails.size()) tails.push_back(envelopes[i][1]);
        else tails[idx] = envelopes[i][1];
    }
    return tails.size();
}
```
*(Note: his submission included a stray `dfs`/`dirs` block copied from the prior problem — dead code, flagged during review.)*

**Optimal Solution:** Same as above — this is the canonical optimal approach.

**Time Complexity:** O(n log n) — sort O(n log n) + tails loop O(n log n) (binary search per element).
**Space Complexity:** O(n) — tails array.

---

## Feedback Given

### Scoring

| Criterion | Score (out of 5) | Notes |
|---|---|---|
| Problem Understanding & Clarification | 4 | Asked for constraints (consistent habit). Handled the tie once highlighted but didn't proactively ask about equal widths — the core trap. |
| Approach & Thought Process | 5 | Excellent. Immediately reduced to sort-one-dimension + LIS-on-other and named the O(n log n) tails method upfront. Structure-exploiting, no O(n^2) fumble. |
| Code Quality & Correctness | 4 | Correct, clean LIS with the right comparator. Lost half a point for leftover dfs/dirs dead code from the prior problem. |
| Complexity Analysis | 4 | Correct final answer O(n log n)/O(n), but first pass undercounted the tails loop as O(n), dropping the log n until pushed. |
| Communication | 4.5 | Concise, accurate. Good lower_bound vs upper_bound articulation; clean, correct dry-run. Terse at times. |

**Overall: 4.3 / 5 — Strong, efficient round.**

### What went well
- Instant recognition of the sort + LIS reduction without prompting — the opposite of the "defaults to generic over structure-exploiting" weakness.
- Clear grasp of lower_bound/strict-LIS and how the descending tie-break interlocks with it.
- Accurate dry-run tracing actual code, not intent.

### What to sharpen
1. Proactively probe the tie case — the equal-width trap is the problem; top candidates surface it themselves. (Recurrence of "doesn't proactively ask about input semantics.")
2. Count the log factor the first time — treat loop x per-iteration cost as automatic. (Most persistent complexity habit.)
3. Start each problem from a clean slate — stale dfs block reads as careless.
