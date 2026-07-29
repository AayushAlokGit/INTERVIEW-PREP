# DSA Round Transcript
**Date:** 2026-07-08
**Start Time:** 11:40
**End Time:** 12:08
**Duration:** 28 minutes
**Problem:** Count of Range Sum
**Topic:** Merge Sort / Divide & Conquer Counting (Prefix Sums)
**Difficulty:** Hard

---

## Problem Statement
Given an integer array `nums` and two integers `lower` and `upper`, return the number of range sums that lie in `[lower, upper]` inclusive.

A range sum `S(i, j)` is the sum of the elements in `nums` between indices `i` and `j` (inclusive), where `i <= j`.

**Example:**
```
Input:  nums = [-2, 5, -1], lower = -2, upper = 2
Output: 3
Explanation: The three range sums in [-2, 2] are:
   S(0,0) = -2
   S(2,2) = -1
   S(0,2) = 2
```

**Constraints:**
- 1 <= nums.length <= 10^5
- -2^31 <= nums[i] <= 2^31 - 1
- -10^5 <= lower <= upper <= 10^5
- The answer is guaranteed to fit in a 32-bit integer.

---

## Conversation Log

**Interviewer:** Presented the problem, example, constraints. Asked for clarifying questions and to keep complexity in mind given n <= 10^5 and the nums[i] range.

**Aayush:** "Brute force is O(n^2) with prefix sums."

**Interviewer:** Correct — P[0..n], S(i,j)=P[j+1]-P[i], count pairs i<j with lower<=P[j]-P[i]<=upper. Asked: (1) can you beat O(n^2)? (2) given nums[i] up to 2^31 and n up to 1e5, anything to be careful about in prefix sums?

**Aayush:** "Overflow, so use long long."

**Interviewer:** Yes — prefix sums reach ~2x10^14. Pushed on the optimization: for each j, count earlier i with P[j]-upper <= P[i] <= P[j]-lower; what technique counts values in a range as you stream?

**Aayush:** Proposed merge sort on the prefix-sum array. After both halves are sorted, count cross pairs: for each i in the first half, find valid j in the second half with lower+P[i] <= P[j] <= upper+P[i]; since the second half is sorted, binary-search the lower and upper bounds.

**Interviewer:** Confirmed — classic merge-sort counting. Cross pairs have i<j automatically; within-half pairs handled by recursion. Asked for precise complexity: cost of counting per merge and total across levels; and whether counting can be linear per merge.

**Aayush:** (1) "O(m log n) just for the counting step during merge." (2) "Maybe we can use the fact that the right half is also sorted to reduce complexity?"

**Interviewer:** Total across all levels = O(n log^2 n). For linear counting, gave the lever: the LEFT half is also sorted, so as P[i] increases the boundaries move monotonically. Asked whether the pointers ever move backward.

**Aayush:** "Once we find the [leftI, leftJ] for the first element... for all subsequent elements the range will only shift forward."

**Interviewer:** Tightened the framing: iterate left half in increasing order; maintain lo/hi pointers into the sorted right half; both only move forward -> O(m) per merge -> O(n log n) overall. Asked for full code; provided C++ scaffold.

**Aayush:** Submitted full solution (below) with two-pointer counting during merge.

**Interviewer:** Confirmed correct and bug-free; noted the key is not resetting lo/hi per i. Asked: (1) justify not resetting (monotonicity), (2) full time and space complexity.

**Aayush:** (1) "Because as P[i] increases, P[j] will also increase in order for the difference to remain in the range." (2) "O(n log n) TC and SC is O(log n) recursion stack."

**Interviewer:** Sharpened the monotonicity statement. Flagged the space answer as incomplete — asked about the prefix array P and the temp vector at the top-level merge.

**Aayush:** "O(n + log n)."

**Interviewer:** Correct — simplifies to O(n); buffers dominate the recursion stack.

---

## Solution
**Aayush's Final Solution (correct, bug-free on first submission):**
```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution {
public:
    long long lo_, up_;
    vector<long long> P;

    long long mergeCount(int l, int r) {
        if (l >= r) return 0;

        int mid = l + (r - l) / 2;
        long long count = mergeCount(l, mid) + mergeCount(mid + 1, r);

        // Count cross pairs (both halves sorted; pointers never retreat)
        int low = mid + 1;
        int high = mid + 1;
        for (int i = l; i <= mid; i++) {
            while (low <= r && P[low] - P[i] < lo_) low++;
            while (high <= r && P[high] - P[i] <= up_) high++;
            count += (high - low);
        }

        // Merge the two sorted halves
        vector<long long> temp;
        int i = l, j = mid + 1;
        while (i <= mid && j <= r) {
            if (P[i] <= P[j]) temp.push_back(P[i++]);
            else              temp.push_back(P[j++]);
        }
        while (i <= mid) temp.push_back(P[i++]);
        while (j <= r)   temp.push_back(P[j++]);
        for (int k = l; k <= r; k++) P[k] = temp[k - l];

        return count;
    }

    int countRangeSum(vector<int>& nums, int lower, int upper) {
        lo_ = lower; up_ = upper;
        int n = nums.size();
        P.assign(n + 1, 0);
        for (int i = 0; i < n; i++) P[i + 1] = P[i] + nums[i];
        return (int) mergeCount(0, n);
    }
};

int main() {
    vector<int> nums = {-2, 5, -1};
    int lower = -2, upper = 2;
    Solution sol;
    cout << sol.countRangeSum(nums, lower, upper) << endl; // 3
    return 0;
}
```

**Optimal Solution (if different):** Same as above — this is the intended `O(n log n)` merge-sort counting solution. (A Fenwick/BIT or balanced-BST over coordinate-compressed prefix sums is an alternative with the same asymptotics.)

**Time Complexity:** O(n log n) — log n merge levels, O(n) two-pointer counting + merge per level.
**Space Complexity:** O(n) — prefix array + temp buffer dominate the O(log n) recursion stack.

---

## Feedback Given

**Time Taken: 28 minutes**

### What went well
- Reached the merge-sort-counting paradigm quickly; framed cross-pair counting correctly on the first try.
- Clean, bug-free code on first submission — two-pointer counting, strict-< / <= boundaries, and merge all correct. Traced against the example, returns 3.
- Caught overflow immediately when pointed at the value range (long long) — the exact scale-awareness that bit him in the previous round; good transfer.
- Nailed the O(n log n) optimization by recognizing both halves are sorted -> monotonic pointers.

### What to work on
1. Space accounting (recurring): first answered O(log n), counting only the recursion stack and forgetting the O(n) prefix array and temp buffer. Corrected to O(n) on prompt. Habit: space = call stack + every auxiliary allocation.
2. State invariants precisely: the monotonicity justification was directionally right but loose. Crisp version: left half sorted => P[i] non-decreasing => both window boundaries non-decreasing => pointers never retreat.

### Scorecard
| Criterion | Score | Notes |
|---|---|---|
| Problem Understanding & Clarification | 4 / 5 | Caught overflow on cue; solid range-sum framing |
| Approach & Thought Process | 4.5 / 5 | Reached merge-sort counting fast; found two-pointer opt |
| Code Quality & Correctness | 4.5 / 5 | Correct, bug-free first submission |
| Complexity Analysis | 3.5 / 5 | Time correct; space undercounted until prompted |
| Communication | 4 / 5 | Clear; invariant justification a touch loose |

**Overall:** Notably stronger than the prior round — especially code correctness (zero bugs) and applying the overflow lesson. Remaining soft spot is complexity precision, specifically space.
