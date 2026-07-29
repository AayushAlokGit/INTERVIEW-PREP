# DSA Round Transcript
**Date:** 2026-05-22
**Start Time:** 9:47
**End Time:** 10:51
**Duration:** 64 minutes
**Problem:** Search in Rotated Sorted Array
**Topic:** Binary Search
**Difficulty:** Medium

---

## Problem Statement
There is an integer array `nums` sorted in ascending order with distinct values. Before being passed to the function, `nums` is rotated at an unknown pivot index `k` so that the array becomes `[nums[k], ..., nums[n-1], nums[0], ..., nums[k-1]]`.

Given the rotated array `nums` and an integer `target`, return the index of `target` if it is in `nums`, or `-1` if it is not.

**Example 1:** nums = [4,5,6,7,0,1,2], target = 0 -> Output: 4
**Example 2:** nums = [4,5,6,7,0,1,2], target = 3 -> Output: -1
**Example 3:** nums = [1], target = 0 -> Output: -1

Constraints: 1 <= nums.length <= 5000; -10^4 <= nums[i] <= 10^4; all values distinct; -10^4 <= target <= 10^4. Expected time complexity: O(log n).

---

## Conversation Log

**Interviewer:** Presented the problem, asked for clarifying questions. Asked Aayush to note start time.

**Aayush:** Start time 9:47.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints (n up to 5000, values within +/-10^4, distinct, target within +/-10^4). Noted the expected complexity should be O(log n) given the sorted-then-rotated structure.

**Aayush:** Find the pivot point, then binary search the two halves around it. Pivot point = nums[0] of the original (pre-rotation) array, i.e. the minimum. Binary search for the pivot: lo=0, hi=n-1. If nums[mid] > nums[0]: pivot not in left half, lo=mid+1. Else if nums[mid] <= nums[0]: pivot=mid, hi=mid-1.

**Interviewer:** Asked Aayush to dry-run the pivot search on [4,5,6,7,0,1,2].

**Aayush:** lo=0 hi=6 mid=3 -> lo=4; lo=4 hi=6 mid=5 -> pivot=5, hi=4; lo=4 hi=4 mid=4 -> pivot=4, hi=3. Returns pivot 4.

**Interviewer:** Correct for that array, but one passing trace isn't proof. Asked Aayush to dry-run the same search on [5,1,2,3,4].

**Aayush:** lo=0 hi=4 mid=2 -> pivot=2, hi=1; lo=0 hi=1 mid=0 -> pivot=0, hi=-1. Returns 0.

**Interviewer:** Asked where the actual minimum is.

**Aayush:** Index 1.

**Interviewer:** Confirmed it's a bug. Asked Aayush to diagnose why index 0 always satisfies the else condition.

**Aayush:** The binary search fails since nums[0] is always <= nums[0], so the algorithm can't find the pivot accurately. Alternative: the pivot is less than both its left and right neighbors; initialize pivot=-1 and update only when that criteria holds; if it stays -1 there is no pivot. Be mindful of mid=0 and mid=n-1 which can never be pivots.

**Interviewer:** The neighbor-check confirms whether mid IS the pivot, but binary search also needs a directional rule for when mid is NOT the pivot. Asked what comparison decides which half holds the pivot.

**Aayush:** If nums[mid] > nums[lo] then [lo:mid] is properly sorted, so lo=mid+1, else hi=mid.

**Interviewer:** Asked Aayush to dry-run the new rule on [5,1,2,3,4].

**Aayush:** hi becomes 0 and lo becomes 0; if pivot is updated by the neighbor condition, pivot would be 1.

**Interviewer:** Explained that relying on the neighbor-check to rescue a buggy directional search is fragile — the search isn't guaranteed to visit the pivot. The anchor nums[lo] fails because when mid==lo, nums[mid] > nums[lo] is false. Hinted: anchor on nums[hi] instead. Asked for the two branches.

**Aayush:** nums[mid] > nums[hi] -> lo=mid+1, else hi=mid.

**Interviewer:** Correct. Asked Aayush to confirm with a trace and write the complete solution; nudged on how to pick which sorted run contains target.

**Aayush:** Requested boilerplate; interviewer provided a 3-step C++ skeleton.

**Aayush:** Submitted full C++ solution (see below).

**Interviewer:** Asked Aayush to dry-run on Example 1 and give the complexity.

**Aayush:** O(log n) time, O(1) space. pivot=4, decides to search for 0 in [pivot, n-1], returns index 4.

**Interviewer:** Asked Aayush to name the edge cases handled: non-rotated array and single-element array.

**Aayush:** pivot=0 handles the non-rotated sorted array; single-element array is also handled by pivot=0.

**Interviewer:** Confirmed solution is complete and correct. Asked for end time.

**Aayush:** End time 10:51.

---

## Solution
**Aayush's Final Solution (C++):**
```cpp
int search(vector<int>& nums, int target) {
    int n = nums.size();

    // ---- Step 1: find pivot (index of the minimum element) ----
    int lo = 0, hi = n - 1;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] > nums[hi])      // mid is left of pivot -> go right
            lo = mid + 1;
        else if (nums[hi] >= nums[mid]) // mid could be pivot; right of mid is not
            hi = mid;
    }
    int pivot = hi;

    // ---- Step 2: pick the sorted run that could contain target ----
    if (target < nums[pivot]) return -1;

    if (pivot == 0) {                  // array already in ascending order
        lo = 0; hi = n - 1;
    } else {
        if (target >= nums[0]) { lo = 0;     hi = pivot - 1; }
        else                   { lo = pivot; hi = n - 1;     }
    }

    // ---- Step 3: plain binary search on that run ----
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (nums[mid] == target) return mid;
        if (nums[mid] > target) hi = mid - 1;
        else                    lo = mid + 1;
    }

    return -1;
}
```

**Optimal Solution:** This is an optimal O(log n) approach (two-pass: find pivot, then binary search the correct run). A common one-pass alternative does a single modified binary search that, at each step, determines which half is sorted and whether target lies within it — also O(log n). Aayush's two-pass version is equally optimal in big-O.

**Time Complexity:** O(log n) — two binary searches.
**Space Complexity:** O(1).

---

## Feedback Given

# Feedback — Search in Rotated Sorted Array (Medium)

**Time Taken: 64 minutes**

## Overall
Reached a correct, optimal O(log n) solution that handles edge cases cleanly. The path there was long (64 minutes), with the bulk spent fighting the pivot-finding binary search. Final code is good; the concern is how many iterations it took to land the search predicate.

## What went well
- Asked for constraints up front, unprompted — a real, visible improvement on the #1 recurring weakness.
- Picked the structure-exploiting binary-search approach rather than a linear scan.
- Step 2 (which run to search) handled independently and correctly — the target >= nums[0] split, pivot==0 non-rotated case, and target < nums[pivot] early exit.
- Sharp diagnosis: correctly identified that the nums[0] anchor breaks because index 0 always satisfies <=.
- Final code: no compile bugs, clean structure, correct.

## What to sharpen
1. Binary search anchor selection cost the round. Went through three anchors — nums[0] with <= (broke monotonicity), nums[lo] (broke when mid==lo), and finally nums[hi] only after a direct hint. Before coding a binary search, explicitly state the predicate and verify it is monotonic (F...F T...T) over the whole range. The anchor must be an element that cannot move into the wrong region — a fixed end (nums[hi]) works, a moving one (nums[lo]) or an always-equal one (nums[0]) does not.
2. Don't rescue a buggy search with a side-check. Catching the pivot via a neighbor-check while the directional search runs wrong relies on luck. A binary search is correct only if its invariant holds every step — fix the invariant, don't patch around it.
3. Trace proactively. Interviewer had to request every dry-run. Volunteer traces on both a rotated and a non-rotated case before declaring done.

## Scoring

| Criterion | Score | Notes |
|---|---|---|
| Problem understanding & clarification | 8/10 | Asked for constraints up front — real improvement |
| Approach & thought process | 6/10 | Right strategy; three wrong anchors before landing it |
| Code quality & correctness | 8/10 | Final code correct, clean, edge cases handled |
| Complexity analysis | 9/10 | O(log n) / O(1), immediate and correct |
| Communication | 7/10 | Good diagnosis; the "rescue" idea showed shaky BS intuition |

**Overall: 38/50** — correct and optimal outcome; the binary-search predicate is the clear thing to drill.
