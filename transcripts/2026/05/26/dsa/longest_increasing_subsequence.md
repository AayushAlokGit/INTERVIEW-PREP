# DSA Round Transcript
**Date:** 2026-05-26
**Start Time:** 10:10
**End Time:** 10:27
**Duration:** 17 minutes
**Problem:** Longest Increasing Subsequence
**Topic:** DP / Binary Search (Patience Sort)
**Difficulty:** Medium

---

## Problem Statement
Given an integer array `nums`, return the length of the **longest strictly increasing subsequence**.

**Examples:**
- `[10,9,2,5,3,7,101,18]` → 4
- `[0,1,0,3,2,3]` → 4
- `[7,7,7,7,7,7,7]` → 1

**Constraints:**
- 1 <= nums.length <= 2500
- -10^4 <= nums[i] <= 10^4

---

## Conversation Log

**Interviewer:** Presented problem.

**Aayush:** Constraints?

**Interviewer:** Provided.

**Aayush:** Brute force — generate all subsequences and check. O(2^n · n).

**Interviewer:** Better?

**Aayush:** O(n²) DP. lis[i] = max(lis[i], lis[j]+1) for j<i with nums[j]<nums[i]. Answer = max of lis array.

**Interviewer:** Confirmed. Asked for TC/SC.

**Aayush:** O(n²), O(n).

**Interviewer:** Better?

**Aayush:** Tails array — tails[i] = best tail for LIS of length i. For each num, binary search for first element >= num, replace; or append if none.

**Interviewer:** Asked: lower_bound or upper_bound for strictly increasing?

**Aayush:** upper_bound.

**Interviewer:** Walked through [7,7,7,7] showing upper_bound gives length 2 instead of 1. Asked again.

**Aayush:** lower_bound, and length of tails is the answer.

**Interviewer:** Correct. Asked to code.

**Aayush:** [submitted clean correct code with lower_bound, push-vs-replace logic]

**Interviewer:** TC/SC?

**Aayush:** O(n log n), O(n).

**Interviewer:** Follow-up — how to reconstruct the actual LIS?

**Aayush:** Parent mapping — when inserting element at idx in tails, the element previously at tails[idx-1] (if any) is the parent.

**Interviewer:** Correct.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> nums{10, 9, 2, 5, 3, 7, 101, 18};
    int n = nums.size();
    vector<int> tails;
    for (int i : nums) {
        int idx = lower_bound(tails.begin(), tails.end(), i) - tails.begin();
        if (idx == (int)tails.size()) {
            tails.push_back(i);
            continue;
        }
        tails[idx] = i;
    }
    cout << "ans is " << tails.size();
    return 0;
}
```

**Time Complexity:** O(n log n)
**Space Complexity:** O(n)

---

## Feedback Given

### Scoring (out of 5)

| Category | Score | Notes |
|---|---|---|
| Problem understanding & clarification | 3.5 / 5 | Asked constraints. Didn't explicitly clarify strict vs non-strict or duplicates. |
| Approach & thought process | 4.5 / 5 | Clean escalation brute → O(n²) DP → O(n log n) patience sort, self-driven. |
| Code quality & correctness | 5 / 5 | First-try clean. Correct lower_bound, correct push-vs-replace. No bugs. |
| Complexity analysis | 5 / 5 | Correct on both DP and patience sort versions. |
| Communication | 4.5 / 5 | Crisp explanation. Good follow-up answer for reconstruction. |

### Highlights
- Crisp DP formulation.
- Knew patience-sort trick cold.
- Got parent-pointer reconstruction follow-up immediately.

### Areas to work on
- Binary search anchor — defaulted to upper_bound for strictly increasing. Mental rule: strictly → lower_bound, non-strictly → upper_bound.
- Should clarify strict vs non-strict up-front.

**Time Taken: 17 minutes**
