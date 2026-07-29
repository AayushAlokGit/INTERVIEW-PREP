# DSA Round Transcript
**Date:** 2026-05-05
**Start Time:** 10:38
**End Time:** 11:16
**Duration:** 38 minutes
**Problem:** Find K Closest Elements
**Topic:** Binary Search, Two Pointers
**Difficulty:** Medium

---

## Problem Statement

Given a **sorted** integer array `arr`, two integers `k` and `x`, return the `k` closest integers to `x` in the array. The result should also be **sorted in ascending order**.

An integer `a` is closer to `x` than an integer `b` if:
- `|a - x| < |b - x|`, or
- `|a - x| == |b - x|` and `a < b`

**Example 1:**
```
Input: arr = [1,2,3,4,5], k = 4, x = 3
Output: [1,2,3,4]
```

**Example 2:**
```
Input: arr = [1,2,3,4,5], k = 4, x = -1
Output: [1,2,3,4]
```

**Example 3:**
```
Input: arr = [1,1,2,3,4,5], k = 4, x = -1
Output: [1,1,2,3]
```

**Constraints:**
- `1 <= k <= arr.length`
- `1 <= arr.length <= 10^4`
- `arr` is sorted in ascending order
- `-10^4 <= arr[i], x <= 10^4`

---

## Conversation Log

**Interviewer:** Presented the problem (initially "Longest Substring with At Most K Distinct Characters", swapped after Aayush said he had solved it). Asked for clarifying questions.

**Aayush:** "x can be out of bound of elements of array or it could be n element of the array or it could lie in the range of the array elements right?"

**Interviewer:** Confirmed all four cases (x < arr[0], x > arr[n-1], x equals an element, x between two elements) are valid.

**Aayush:** "A brute force solution would involve calculating abs(x-arr[i]) with each element and storing the element as {abs(x-arr[i]),arr[i]} in a vector of pairs and then sort this vector firstly on the basis of abs(x-arr[i]) and then on arr[i] return the first k elements. O(n log n) time, O(n) space."

**Interviewer:** Acknowledged correct, mentioned heap variant for O(n log k). Pushed for better given sorted input.

**Aayush:** "We could get the element in array just >= x, set two pointers at this element and then move to the left or right greedily depending on which move would give a better answer."

**Interviewer:** Confirmed, sketched binary search + two-pointer expansion. Asked about tiebreak and bounds.

**Aayush:** "Will k be < or > the array size?"

**Interviewer:** Pointed to constraints: 1 <= k <= arr.length.

**Aayush:** "If there is only one element in the array then return the array right?"

**Interviewer:** Yes — k must be 1 in that case.

**Aayush:** Submitted first version with bugs: typo `leftDiffDiff`, evaluated `arr[l-1]/arr[r+1]` but pushed `arr[l]/arr[r]` causing double-push, loop condition `&&` exits early when one side exhausted, result not sorted.

**Interviewer:** Walked through trace on `[1,2,3,4,5], k=4, x=3` showing `3` would be pushed twice. Asked four questions about the bugs.

**Aayush:** Submitted second version: changed initial state to `l=start-1, r=start+1`, pushed `arr[start]` upfront, kept evaluating `arr[l-1]`/`arr[r+1]`, left `&&` in loop, sorted at end.

**Interviewer:** Pointed out:
1. Index mismatch — evaluating `arr[l-1]` but pushing `arr[l]` — pick one convention.
2. Bounds check `(l>=0)? arr[l-1]` is OOB when `l=0`.
3. Loop condition `&&` exits early.
4. Sort at end is `O(k log k)` — answer is contiguous, just track `lo`/`hi`.

**Aayush:** Submitted third version: corrected to evaluate `arr[l]/arr[r]`, fixed bounds, changed loop to `||`, removed sort and used final `for(i=l+1;i<r;i++)` to populate ans. But left earlier `ans.push_back(arr[start])` resulting in duplicate `3` in `ans` vector (output looked correct because only the second loop was printed).

**Interviewer:** Pointed out the leftover `ans.push_back(arr[start])` causing duplicate. Asked for the binary-search-on-window optimization.

**Aayush:** "If x-arr[lo] and arr[lo+k]-x have same sign, then if sign is +ve move lo to left else move lo to right. If they have different signs => x lies somewhere in between the k sized window. Then if abs(x-arr[lo]) > abs(x-arr[lo+k]) then move lo to right else left?"

**Interviewer:** Said overcomplicated. Showed simpler unified condition: compare `x - arr[mid]` vs `arr[mid+k] - x`; if first larger move right, else move left. Works for all sign cases due to sorted array.

**Aayush:** "11:16"

---

## Solution

**Aayush's Final Solution (two-pointer expansion, with leftover bug):**
```cpp
start = lower_bound(arr.begin(), arr.end(), x) - arr.begin();
int l = start - 1, r = start + 1;
ans.push_back(arr[start]);  // BUG: leftover, causes duplicate
int cnt = 1;
while ((l >= 0 || r < n) && cnt < k) {
    int leftDiff = (l >= 0) ? abs(arr[l] - x) : INT_MAX;
    int rightDiff = (r < n) ? abs(arr[r] - x) : INT_MAX;
    if ((leftDiff == rightDiff && leftDiff != INT_MAX) || leftDiff < rightDiff) {
        l--; cnt++;
    } else if (rightDiff < leftDiff) {
        r++; cnt++;
    }
}
for (int i = l + 1; i < r; i++) ans.push_back(arr[i]);
```

**Optimal Solution (binary search on window left boundary):**
```cpp
vector<int> findClosestElements(vector<int>& arr, int k, int x) {
    int n = arr.size();
    int lo = 0, hi = n - k;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (x - arr[mid] > arr[mid + k] - x) lo = mid + 1;
        else hi = mid;
    }
    return vector<int>(arr.begin() + lo, arr.begin() + lo + k);
}
```

**Time Complexity:** O(log(n-k) + k) — binary search + slice copy
**Space Complexity:** O(1) extra (excluding output)

Aayush's solution: O(log n + k) time, O(1) extra space.

---

## Feedback Given

### Scoring

| Category | Score (1-5) | Notes |
|---|---|---|
| Problem Understanding & Clarification | 4 | Asked good edge-case questions (x out of range, k vs n, single element). Could have clarified array size constraints upfront. |
| Approach & Thought Process | 4 | Brute force → heap → two-pointer expansion progression was solid. Recognized the sorted-array hint immediately. Didn't reach binary-search-on-window approach independently. |
| Code Quality & Correctness | 2.5 | Multiple bugs across iterations: typo (leftDiffDiff), index/diff mismatch, bounds check off-by-one, && vs \|\| in loop, leftover ans.push_back duplicate. Most bugs only surfaced when prompted to trace. |
| Complexity Analysis | 4 | Correctly identified O(n log n) brute force, O(log n + k) two-pointer. Didn't proactively offer space complexity. |
| Communication | 4 | Clear thinking out loud, articulated edge cases well, responded constructively to hints. |

**Overall: 3.5 / 5**

### Time Taken: 38 minutes

### Key Takeaways

1. **Trace before submitting.** Three iterations of the same code, each had a different bug discoverable by running through `arr=[1,2,3,4,5], x=3, k=4` in your head.
2. **Bounds checks in C++ must match the access pattern.** `(l>=0)? arr[l-1]` is wrong — the bound must guard the actual index accessed.
3. **Clean up when switching approaches.** The leftover `ans.push_back(arr[start])` silently corrupted the result. Recurring pattern.
4. **Look for "the answer is a contiguous window" insights.** When you see this structure, binary-searching on the window's left boundary is often O(log n + k) and cleaner than expanding around a pivot.
