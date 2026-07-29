# DSA Round Transcript
**Date:** 2026-06-15
**Start Time:** 9:43
**End Time:** 10:32
**Duration:** 49 minutes
**Problem:** Find First and Last Position of Element in Sorted Array
**Topic:** Binary Search (Lower/Upper Bound)
**Difficulty:** Medium

---

## Problem Statement
Given an array of integers `nums` sorted in non-decreasing order, find the starting and ending position of a given `target` value. Return `[first, last]`. If `target` is not found, return `[-1, -1]`.

**Example 1:** `nums = [5,7,7,8,8,10], target = 8` → `[3,4]`
**Example 2:** `nums = [5,7,7,8,8,10], target = 6` → `[-1,-1]`
**Example 3:** `nums = [], target = 0` → `[-1,-1]`

**Constraints:**
```
0 <= nums.length <= 10^5
-10^9 <= nums[i] <= 10^9
-10^9 <= target <= 10^9
nums sorted non-decreasing
```

---

## Conversation Log

**Interviewer:** Presented the problem; asked for clarifying questions.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints; noted empty array and negatives possible. Asked what time complexity to target given the array is sorted.

**Aayush:** What if there is only one instance of target in the array?

**Interviewer:** Then first == last, return [i, i]. Asked for target complexity and approach.

**Aayush:** Use binary search — two binary searches:
1. First index >= target (lower bound): if `nums[mid] >= target`, `hi = mid`, else `lo = mid+1`.
2. First index > target (upper bound).
If target present, `index1 < n` and `nums[index1] == target`; result range is `[index1, index2-1]`.

**Interviewer:** Solid — both O(log n). Code it carefully.

**Aayush:** (submitted C++ — two half-open binary searches, presence check, see below)

**Interviewer:** Dry-run `nums=[], target=0` and `nums=[5,7,7,8,8,10], target=11`. Also: does the order of `left == nums.size() || nums[left] != target` matter?

**Aayush:** Order matters — if `left == n` and we access the index we'd get an out-of-bounds error, so the bounds check must come first. `[], target=0` → `{-1,-1}` since left==0==size. `target=11` → `{-1,-1}` since left==6.

**Interviewer:** Correct on all three. Complexity?

**Aayush:** O(log n) time, O(1) space.

**Interviewer:** Worst case of the naive "find one occurrence then expand linearly" approach, and on what input?

**Aayush:** O(log n) to find the index + O(n) to expand boundaries, because the whole array could be the target element.

**Interviewer:** Correct — done. Clean optimal solve.

---

## Solution
**Aayush's Final Solution:**
```cpp
int binarySearch1(vector<int>& nums, int target) {   // first index >= target
    int lo = 0, hi = nums.size();
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] >= target) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}

int binarySearch2(vector<int>& nums, int target) {   // first index > target
    int lo = 0, hi = nums.size();
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] > target) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}

// main:
int left  = binarySearch1(nums, target);
int right = binarySearch2(nums, target);
if (left == nums.size() || nums[left] != target)
    return {-1, -1};
else
    return {left, right - 1};
```

**Optimal Solution:** Same — this is optimal.
**Time Complexity:** O(log n)
**Space Complexity:** O(1)

---

## Feedback Given

### Strengths
- Excellent clarification: asked constraints AND the single-instance edge case proactively.
- Went straight to the optimal lower-bound/upper-bound approach.
- Clean, correct binary search with consistent half-open intervals — no off-by-one (a notable improvement over a recurring weakness).
- Reasoned about safety: explained why the short-circuit order protects against out-of-bounds access.
- Airtight complexity, plus correctly diagnosed the naive expand-from-center approach as O(n) on an all-target array.

### Areas to work on
- Pace: 49 minutes is long for a problem solved without false starts; aim for ~20-25 min on a medium like this.
- Self-initiated verification: code was correct, but he still waited to be prompted for the edge-case trace rather than volunteering it. Make the trace a reflex before declaring done.

### Scoring (out of 5)
| Criterion | Score | Note |
|---|---|---|
| Problem understanding & clarification | 4.5 | Constraints + single-instance edge case, proactively |
| Approach & thought process | 4.5 | Optimal lower/upper bound immediately |
| Code quality & correctness | 4.5 | Clean binary search, no off-by-one |
| Complexity analysis | 5.0 | Correct + correctly analyzed naive alternative |
| Communication | 4.5 | Clear, reasoned about safety, traced accurately |

**Time Taken: 49 minutes**
