# DSA Round Transcript
**Date:** 2026-06-29
**Start Time:** 11:01
**End Time:** 11:29
**Duration:** 28 minutes
**Problem:** Find Minimum in Rotated Sorted Array II
**Topic:** Binary Search (rotated array with duplicates)
**Difficulty:** Medium/Hard

---

## Problem Statement
An array of length `n` sorted in ascending order is rotated between `1` and `n` times. The rotated array `nums` **may contain duplicates**. Return the minimum element.

**Example 1:** `nums = [1, 3, 5]` -> `1`
**Example 2:** `nums = [2, 2, 2, 0, 1]` -> `0`

**Constraints:**
- `1 <= n <= 5000`
- `-5000 <= nums[i] <= 5000`
- sorted ascending (possibly with duplicates) then rotated 1..n times.

---

## Conversation Log

**Interviewer:** Presented problem and examples, asked for clarifying questions.

**Aayush:** What are the constraints?

**Interviewer:** Provided constraints (n up to 5000, values -5000..5000, may contain duplicates).

**Aayush:** Brute force is one pass finding min in O(n).

**Interviewer:** Works as baseline, but the array is sorted+rotated — can you do better? Target complexity?

**Aayush:** Binary search for the pivot element, which is the minimum.

**Interviewer:** Asked for the comparison logic (anchor choice) and how duplicates are handled; asked him to walk `[2,2,2,0,1]` and `[2,2,0,2,2]`.

**Aayush:** Compare mid with hi (comparison with lo is mostly ambiguous). If nums[mid] > nums[hi] then lo = mid+1, else hi = mid; mid could be the pivot.

**Interviewer:** Confirmed correct anchor and `hi = mid` (not mid-1). Asked him to code it, being deliberate about the `==` case folding into `else`.

**Aayush:** Submitted first version where `==` folds into `else { hi = mid; }`.

**Interviewer:** Asked him to dry-run on `[3,3,1,3,3,3,3]` (valid rotation of `[1,3,3,3,3,3,3]`, answer = 1), tracing every iteration.

**Aayush:** Traced it: lo0/hi6/mid3 -> hi=3; lo0/hi3/mid1 -> hi=1; lo0/hi1/mid0 -> hi=0; returns nums[0]=3. Found the bug — returns 3 instead of 1.

**Interviewer:** Confirmed bug is the `==` case. Asked (1) can you determine which side the min is on when nums[mid]==nums[hi]? (2) safe action and complexity cost?

**Aayush:** (1) Can't determine which side. (2) Do hi-- because nums[hi] is guaranteed not to be the unique minimum — an equal element exists in [lo, hi-1].

**Interviewer:** Confirmed the safety argument. Asked worst-case complexity and trigger, plus average case.

**Aayush:** Worst case O(n) when all elements are equal.

**Interviewer:** Confirmed (avg O(log n), space O(1)). Asked for the final corrected loop.

**Aayush:** Submitted corrected three-way version.

**Interviewer:** Verified on `[3,3,1,3,3,3,3]`->1, `[2,2,2,0,1]`->0, `[1,3,5]`->1, `[2,2,2]`->2, `[5]`->5. All correct.

---

## Solution
**Aayush's Final Solution (correct):**
```cpp
int findMin(vector<int>& nums) {
    int lo = 0, hi = nums.size() - 1;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] > nums[hi]) {
            lo = mid + 1;
        } else if (nums[mid] < nums[hi]) {
            hi = mid;
        } else {           // nums[mid] == nums[hi]: can't decide, shrink safely
            hi--;
        }
    }
    return nums[lo];   // lo == hi
}
```

**Optimal Solution:** Same as above — this is the optimal approach.

**Time Complexity:** Average O(log n); worst case O(n) when all/most elements are equal.
**Space Complexity:** O(1).

---

## Feedback Given

**What went well (clear improvement over previous round)**
- Strong binary-search reasoning: correct anchor (nums[hi], not nums[lo]) with the right justification, and kept hi = mid (mid is a live candidate) rather than mid-1.
- Self-verified and found his own bug: traced `[3,3,1,3,3,3,3]`, saw it return 3, and diagnosed the `==`-into-`else` flaw himself.
- Excellent justification for hi--: even if hi were the min, an equal copy survives in [lo, hi-1] via mid, so nothing is lost.
- Complexity sharp and complete without prompting: worst O(n) (all-equal), avg O(log n), space O(1).

**What to keep working on**
- The dry-run was reactive, not proactive — shipped the buggy version as "done," found the bug only when asked to trace. Should self-test the adversarial case (heavy duplicates — the headline of the problem) before declaring done.
- Clarification: asked for constraints but didn't proactively flag duplicates as the crux even though it is the whole twist of this variant.

### Scoring (out of 5)
| Criterion | Score | Notes |
|---|---|---|
| Problem understanding & clarification | 3.5 | Asked constraints; didn't proactively flag duplicates as the crux. |
| Approach & thought process | 4.5 | Brute -> binary search -> correct anchor, cleanly reasoned. |
| Code quality & correctness | 3.5 | First version had the `==` bug; found and fixed it correctly via self-trace. |
| Complexity analysis | 4.5 | Worst O(n) all-equal, avg O(log n), space O(1) — all correct, unprompted. |
| Communication | 4.5 | Clear, well-justified reasoning throughout. |

**Overall:** Noticeably stronger round. Solid binary-search fundamentals; self-verification kicked in when needed. Push it one step earlier — test the obvious adversarial case before declaring done — for a clean pass.

**Time Taken: 28 minutes**
