# DSA Round Transcript
**Date:** 2026-06-07
**Start Time:** 17:24
**End Time:** 17:39
**Duration:** 15 minutes
**Problem:** Partition Equal Subset Sum
**Topic:** Dynamic Programming (0/1 Knapsack / Subset Sum)
**Difficulty:** Medium

---

## Problem Statement
Given an integer array `nums`, return `true` if you can partition the array into two subsets such that the sum of the elements in both subsets is equal. Otherwise, return `false`. Every element must belong to exactly one of the two subsets.

**Example 1:**
```
Input:  nums = [1, 5, 11, 5]
Output: true
Explanation: The array can be partitioned as [1, 5, 5] and [11].
```

**Example 2:**
```
Input:  nums = [1, 2, 3, 5]
Output: false
Explanation: The array cannot be partitioned into two equal-sum subsets.
```

**Constraints:**
- `1 <= nums.length <= 200`
- `1 <= nums[i] <= 100`

---

## Conversation Log

**Interviewer:** Presented the problem and asked for clarifying questions.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints (length 1–200, values 1–100, all positive, max total sum 20,000).

**Aayush:** This problem boils down to finding if there exists a subset in the array with sum = (sum of all elements)/2. This can be solved using DP.

**Interviewer:** Correct reduction. Asked: is there anything to check about the division before starting the DP? And asked him to flesh out state/recurrence/dimensions.

**Aayush:** If total sum is odd then not possible.

**Interviewer:** Confirmed — odd total → return false immediately. Asked for full DP state and recurrence.

**Aayush:**
- dp[i][j] = can we get sum j from i elements of nums
- dp[0][j] = 0
- dp[i][0] = 1
- j <= sum/2, i <= n
- dp[i][j] |= dp[i-1][j-nums[i]] if j >= nums[i]
- dp[i][j] |= dp[i-1][j]

**Interviewer:** Logic correct. Flagged dp[0][0] base-case conflict and the nums[i] vs nums[i-1] indexing concern. Asked him to write the code.

**Aayush:** (Wrote 1D space-optimized C++ solution — see below.) Said TC is O(n*sum) and SC is same.

**Interviewer:** Clean 1D solution, correct reverse iteration. Pushed on space complexity — how big is the dp array, does it grow with n?

**Aayush:** Sorry, O(n).

**Interviewer:** Still not right — is target bounded by n or something else?

**Aayush:** target.

**Interviewer:** Correct — O(target) = O(sum). Asked what edge cases he'd test and to trace the trickiest.

**Aayush:** Odd sum element, and only 1 element in array which is even.

**Interviewer:** Asked him to actually trace nums = [2].

**Aayush:** total=2, target=1, dp=[T,F], dp[1] = F.

**Interviewer:** Correct trace. Is false the right answer, and why?

**Aayush:** Because it would be only one subset.

**Interviewer:** Correct. Asked if there's a way to make it faster exploiting structure (hinted at a data structure).

**Aayush:** Not able to get it.

**Interviewer:** Revealed the bitset optimization (dp |= dp << num, O(n*sum/64)). Wrapped up.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <iostream>
#include <vector>
#include <numeric>

using namespace std;

bool canPartition(vector<int>& nums) {
    int total = accumulate(nums.begin(), nums.end(), 0);

    if (total % 2 != 0)
        return false;

    int target = total / 2;

    vector<bool> dp(target + 1, false);
    dp[0] = true;

    for (int num : nums) {
        for (int s = target; s >= num; --s) {
            dp[s] = dp[s] || dp[s - num];
        }
    }

    return dp[target];
}

int main() {
    vector<int> nums = {1, 5, 11, 5};

    if (canPartition(nums))
        cout << "True\n";
    else
        cout << "False\n";

    return 0;
}
```

**Optimal Solution (bitset optimization — same asymptotic class, ~64x faster in practice):**
```cpp
bitset<10001> dp;
dp[0] = 1;
for (int num : nums)
    dp |= (dp << num);
return dp[target];
```

**Time Complexity:** O(n × sum) — correct
**Space Complexity:** O(target) = O(sum) — reached after two incorrect attempts (said "same" / O(n×sum), then O(n))

---

## Feedback Given

**Overall:** Strong, efficient round. Nailed the core insight fast, wrote correct optimized code on the first pass, traced the edge case accurately when asked. Soft spot was complexity precision (space).

### Scoring

**1. Problem Understanding & Clarification — 5/5**
Proactively asked for constraints first — a clear improvement over the usual pattern of skipping this. Also caught the odd-sum early-exit himself.

**2. Approach & Thought Process — 5/5**
Immediately reduced "two equal subsets" → "subset summing to total/2." DP state, recurrence, and base cases all correct before writing code. Jumped straight to the space-optimized 1D array.

**3. Code Quality & Correctness — 5/5**
Clean, idiomatic C++. Got the critical reverse-iteration detail (s from target downward) right without prompting. Correct as written.

**4. Complexity Analysis — 3/5**
Time correct (O(n × sum)). Space tripped him up: said "same" (O(n×sum)), then "O(n)" — both wrong — before landing on O(target). dp array is sized by target, bounded by n × maxVal, not n. Recurring pattern: be deliberate about what each dimension is bounded by before stating complexity.

**5. Communication — 4/5**
Trace of [2] correct, justified why false is right. Answers very terse; didn't reach the bitset optimization (not heavily penalized — advanced).

**Time Taken: 15 minutes** — efficient pace for a medium DP.

### Top takeaway
DP fundamentals solid and fast. Drill: state complexity by reasoning about each dimension's true bound, not by reflex. "1D array sized target, target ≤ n×100, so space is O(target)" — say it that way every time.
