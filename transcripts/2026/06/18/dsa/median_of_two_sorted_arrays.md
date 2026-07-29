# DSA Round Transcript
**Date:** 2026-06-18
**Start Time:** 11:13
**End Time:** 12:06
**Duration:** 53 minutes
**Problem:** Median of Two Sorted Arrays
**Topic:** Binary Search
**Difficulty:** Hard

---

## Problem Statement
Given two sorted arrays `nums1` and `nums2` of sizes `m` and `n`, return the median of the two sorted arrays combined.

**Examples:**
```
nums1 = [1,3], nums2 = [2]   -> 2.0   (merged [1,2,3])
nums1 = [1,2], nums2 = [3,4] -> 2.5   (merged [1,2,3,4], (2+3)/2)
nums1 = [],   nums2 = [1]    -> 1.0
```

**Constraints:**
- `0 <= m, n <= 1000`, `1 <= m + n <= 2000`
- `-10^6 <= nums1[i], nums2[i] <= 10^6`
- Both arrays sorted non-decreasing.

---

## Conversation Log

**Interviewer:** Presented problem, asked for clarifying questions.
**Aayush:** "what are the constraints?"
**Interviewer:** Gave constraints; asked target complexity and ideal runtime.
**Aayush:** "brute force is merge both arrays (merge step of merge sort) and find median, O(m+n)."
**Interviewer:** Confirmed O(m+n) time (O(1) extra with two pointers). Asked for O(log(m+n)) / O(log(min(m,n))).
**Aayush:** Proposed binary search on the median VALUE in [min, max]. Invariant: smallest value v with count(<=v) >= (n1+n2+1)/2 is the k-th smallest. Odd: that value is the median. Even: that value is the lower middle; also find the (k+1)-th smallest, average the two. Gave the lo/hi binary search skeleton.
**Interviewer:** Confirmed approach valid/clever. Asked for the EXACT time complexity (both factors).
**Aayush:** "f(mid) is log(max(m,n)) via upper_bound; outer loop is log(max_element)."
**Interviewer:** Confirmed O(log V · log(max(m,n))), not strictly O(log(m+n)); good that he didn't overclaim. Noted textbook O(log(min(m,n))) partition method to revisit. Asked him to code, watching boundaries and even-case averaging.
**Aayush:** Wrote code. Empty-array handling moved an element between arrays; computed lo/hi via min_element/max_element; three near-identical binary-search blocks for odd / preMedian / postMedian.
**Interviewer:** Asked him to trace his own test nums1={}, nums2={2}: what do the arrays contain after the if-block, and what does min_element(nums2) do?
**Aayush:** Fixed the code — replaced element-moving with direct globalMn/globalMx from the non-empty array (no mutation of inputs).
**Interviewer:** Confirmed fix correct (globalMn=globalMx=2, loop skipped, median=2). Asked him to trace nums1=[-3], nums2=[-2] (negative values) through the preMedian loop.
**Aayush:** "infinite loop, mid stays at -2. Better version: decrease hi to mid-1, and when condition satisfies store mid in a separate answer variable."
**Interviewer:** Confirmed correct diagnosis. Gave two robust fixes: (1) his store-candidate form with while(lo<=hi), hi=mid-1 / lo=mid+1; (2) one-line fix mid = lo + (hi-lo)/2 (floors toward lo, so hi=mid shrinks). Then revealed the O(log(min(m,n))) partition approach. Wrapped up.

---

## Solution
**Aayush's Final Solution (value binary search, O(log V · log(max(m,n)))) — with agreed fixes:**
```cpp
int countLE(vector<int>&a, vector<int>&b, int x){
    return (upper_bound(a.begin(),a.end(),x)-a.begin())
         + (upper_bound(b.begin(),b.end(),x)-b.begin());
}
// globalMn/globalMx taken from non-empty array(s), no input mutation.
// kth smallest via binary search on value, robust mid:
int kth(vector<int>&a, vector<int>&b, int lo, int hi, int k){
    int ans = hi;
    while (lo <= hi){
        int mid = lo + (hi - lo)/2;          // floors toward lo (negative-safe)
        if (countLE(a,b,mid) >= k){ ans = mid; hi = mid - 1; }
        else lo = mid + 1;
    }
    return ans;
}
// total odd  -> median = kth(.., k);  k=(m+n+1)/2
// total even -> median = (kth(.., k) + kth(.., k+1)) / 2.0;
```

**Optimal Solution (revealed) — partition binary search, O(log(min(m,n))):**
```
Binary search cut1 in [0, m] on the smaller array.
cut2 = (m+n+1)/2 - cut1.
L1 = cut1>0 ? nums1[cut1-1] : -INF;  R1 = cut1<m ? nums1[cut1] : +INF;
L2 = cut2>0 ? nums2[cut2-1] : -INF;  R2 = cut2<n ? nums2[cut2] : +INF;
if (L1 <= R2 && L2 <= R1): correct ->
    odd  -> max(L1,L2)
    even -> (max(L1,L2) + min(R1,R2)) / 2.0
else if (L1 > R2): hi = cut1 - 1; else lo = cut1 + 1;
```

**Bugs found and fixed during the session:**
1. Empty-array handling moved an element and emptied the OTHER array -> UB on min_element. Fixed by reading globalMn/globalMx from the non-empty array directly.
2. `mid = (lo+hi)/2` truncates toward zero -> infinite loop on negative ranges. Fixed via mid = lo + (hi-lo)/2 (or store-candidate + hi=mid-1).
3. (Quality) three duplicated binary-search blocks -> factor into a kth-smallest helper.

**Time Complexity:** O(log V · log(max(m,n))) (his); O(log(min(m,n))) (optimal partition).
**Space Complexity:** O(1).

---

## Feedback Given

**Scoring**

1. **Problem Understanding & Clarification — 4/5** — Asked constraints and engaged with target complexity. Did not proactively flag the empty-array and negative-value edge cases (both explicit in constraints, both broke the code). Build the habit of reading constraints adversarially.

2. **Approach & Thought Process — 4.5/5** — Independently derived a correct, non-obvious value-binary-search with the right k-th-smallest invariant and odd/even handling. Didn't reach the partition method but his approach is valid; correctly acknowledged its value-range dependence.

3. **Code Quality & Correctness — 2.5/5** — Two boundary bugs, neither caught by his own test: empty-array handling reintroduced UB by emptying the other array; (lo+hi)/2 infinite-loops on negative ranges. Both fixed correctly when prompted; infinite-loop diagnosis was sharp. Significant code duplication (three near-identical loops) — should factor into a kth-smallest helper.

4. **Complexity Analysis — 5/5** — Nailed O(log V · log(max(m,n))) with both factors and did not overclaim O(log(m+n)). Improving session over session.

5. **Communication — 4/5** — Clear; on the negative-value input he traced the actual loop, predicted the stall, and proposed a working fix — improvement over earlier "trace intent not code" pattern.

**Overall:** High algorithmic ceiling — derived a correct sublinear approach to a hard problem from scratch and analyzed it precisely. Bottleneck remains boundary execution and self-testing: across three rounds this week every bug lived at an edge (empty, negatives, off-by-one) and every one was hidden by a friendly self-chosen test. Highest-value habit: before declaring done, run hostile inputs first — empty / single element / negatives / all-equal / answer-at-extreme.

**Time Taken: 53 minutes**
