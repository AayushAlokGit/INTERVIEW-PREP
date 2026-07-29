# DSA Round Transcript
**Date:** 2026-05-17
**Start Time:** 9:51
**End Time:** 10:20
**Duration:** 29 minutes
**Problem:** Split Array Largest Sum
**Topic:** Binary Search on Answer
**Difficulty:** Hard

---

## Problem Statement
Given an integer array `nums` and integer `k`, split `nums` into `k` non-empty subarrays such that the largest sum among them is minimized. Return that minimized largest sum.

**Examples:**
- `nums=[7,2,5,10,8], k=2` → `18`
- `nums=[1,2,3,4,5], k=2` → `9`
- `nums=[1,4,4], k=3` → `4`

**Constraints:** `1 <= n <= 1000`, `0 <= nums[i] <= 10^6`, `1 <= k <= min(50, n)`

---

## Conversation Log

**Interviewer:** Presented problem. (Skipped LIS and Min Window Substring on Aayush's request.)

**Aayush:** "Binary search on the answer. For each candidate max_sum, count partitions where each subarray <= max_sum. If partitions < k, reduce max_sum. If == k, candidate; reduce. If > k, increase max_sum."

**Interviewer:** Noted the `<` and `==` cases collapse — both feasible (can pad with extra splits). Asked for lo/hi.

**Aayush:** "lo = 0, hi = sum(nums). Greedy partitioning: extend current subarray while sum + nums[i] <= max_sum, else start new subarray."

**Interviewer:** Suggested tighter lower bound. Asked about single element > max_sum edge case.

**Aayush:** "lo = max(nums). If any element > max_sum, return infinity to force max_sum up." (Naturally avoided by lo = max.)

**Aayush:** Wrote C++ code.

**Interviewer:** Asked him to trace `count` on `[7,2,5,10,8], max_sum=18`.

**Aayush:** "Should be cnt + 1." (Off-by-one — final subarray uncounted.)

**Interviewer:** Two more issues — `ans` uninitialized, and `ans = min(ans, mid)` redundant.

**Aayush:** "We always place hi as a candidate answer, so when loop ends printing hi is enough."

**Interviewer:** Confirmed. Asked complexity.

**Aayush:** "O(n * log(sum)) time, O(1) space."

---

## Solution
**Aayush's Final Solution (with discussed fixes):**
```cpp
int count(vector<int>& nums, int max_sum) {
    int cnt = 0;
    int sum = 0;
    for (int i : nums) {
        if (sum + i > max_sum) { sum = i; cnt++; continue; }
        sum += i;
    }
    return cnt + 1;
}

int splitArray(vector<int>& nums, int k) {
    int sum = 0;
    for (int i : nums) sum += i;
    int lo = *max_element(nums.begin(), nums.end());
    int hi = sum;
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (count(nums, mid) <= k) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}
```

**Optimal Solution (same approach, cleaner with long long for overflow safety):**
```cpp
int splitArray(vector<int>& nums, int k) {
    auto feasible = [&](long long maxSum) {
        int parts = 1;
        long long cur = 0;
        for (int x : nums) {
            if (cur + x > maxSum) { parts++; cur = x; }
            else cur += x;
        }
        return parts <= k;
    };
    long long lo = *max_element(nums.begin(), nums.end());
    long long hi = accumulate(nums.begin(), nums.end(), 0LL);
    while (lo < hi) {
        long long mid = lo + (hi - lo) / 2;
        if (feasible(mid)) hi = mid;
        else lo = mid + 1;
    }
    return (int)lo;
}
```

**Time Complexity:** O(n log S) where S = sum(nums)
**Space Complexity:** O(1)

---

## Feedback Given

### Scoring

| Category | Score (/5) | Notes |
|---|---|---|
| Problem Understanding & Clarification | 2.5 | No clarifying questions. |
| Approach & Thought Process | 4.5 | Identified pattern instantly. |
| Code Quality & Correctness | 2.5 | Three bugs: off-by-one in count, uninitialized ans, redundant min. |
| Complexity Analysis | 5 | Correct. |
| Communication | 4 | Clear, but no proactive edge cases. |

### Key takeaways
1. Self-trace before declaring done — off-by-one in `count` would be caught immediately on the given example.
2. Binary-search-on-answer pattern is solid; adjacent: *Koko Eating Bananas*, *Capacity To Ship Packages*, *Min Days to Make M Bouquets*.

**Time Taken: 29 minutes**
