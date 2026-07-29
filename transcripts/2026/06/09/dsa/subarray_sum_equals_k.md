# DSA Round Transcript
**Date:** 2026-06-09
**Start Time:** 9:34
**End Time:** 9:41
**Duration:** 7 minutes
**Problem:** Subarray Sum Equals K
**Topic:** Prefix Sum + Hash Map
**Difficulty:** Medium

---

## Problem Statement
Given an integer array `nums` and an integer `k`, return the total number of subarrays whose sum equals `k`. A subarray is a contiguous, non-empty sequence of elements within the array.

**Example 1:**
```
Input:  nums = [1, 1, 1], k = 2
Output: 2
Explanation: The subarrays [1,1] (indices 0-1) and [1,1] (indices 1-2) both sum to 2.
```

**Example 2:**
```
Input:  nums = [1, 2, 3], k = 3
Output: 2
Explanation: [1,2] and [3].
```

**Constraints:**
- `1 <= nums.length <= 2 * 10^4`
- `-1000 <= nums[i] <= 1000`
- `-10^7 <= k <= 10^7`

---

## Conversation Log

**Interviewer:** Presented the problem and asked for clarifying questions / approach.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints (length up to 2*10^4, nums[i] in [-1000, 1000], k in [-10^7, 10^7]). Asked what stands out.

**Aayush:** I can use prefix sum with a count map. In one pass we calculate the prefix up to the current index, subtract k from it, and add the frequency of the resultant sum to the answer.

**Interviewer:** Good — prefix sum + count map is the right structure (avoids the sliding-window trap given negatives). Asked: (1) how is the map initialized, (2) is the lookup before or after recording the current prefix, and why does order matter?

**Aayush:** {0:1}; lookup before recording.

**Interviewer:** Asked why — what goes wrong if you record first then look up, especially when k=0?

**Aayush:** If k=0 it would count itself, which is wrong.

**Interviewer:** Correct. Asked him to write the full solution.

**Aayush:** (submitted C++ solution below); TC O(n), SC O(n).

**Interviewer:** Asked him to (1) dry-run example 1 index by index, and (2) reconsider whether unordered_map operations are truly O(1) — what's the worst case?

**Aayush:**
```
i=0, ans=0, prefix=1, map={0:1}{1:1}
i=1, sum=2, ans=1, map={0:1}{1:1}{2:1}
i=2, sum=3, ans=2, map={0:1}{1:1}{2:1}{3:1}
```

**Interviewer:** Trace correct. Pressed on complexity worst case — "O(n) for what?"

**Aayush:** O(n).

**Interviewer:** Clarified: a single unordered_map op is O(n) worst case (collisions) → overall O(n^2) worst case; quote "O(n) average, O(n^2) worst case." Asked for a way to guarantee a hard bound.

**Aayush:** Use an ordered map → O(log n) for insert and checking.

**Interviewer:** Correct — std::map gives guaranteed O(n log n) overall. Solution is optimal for expected time. Moved to feedback.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <iostream>
#include <vector>
#include <unordered_map>
using namespace std;

int subarraySum(vector<int>& nums, int k) {
    unordered_map<int, int> prefixCount;
    prefixCount[0] = 1;  // Empty prefix

    int prefixSum = 0;
    int count = 0;

    for (int num : nums) {
        prefixSum += num;

        // Check if there exists a previous prefix sum
        // such that currentSum - previousSum = k
        if (prefixCount.find(prefixSum - k) != prefixCount.end()) {
            count += prefixCount[prefixSum - k];
        }

        prefixCount[prefixSum]++;
    }

    return count;
}
```
**Optimal Solution (if different):** Same — this is the optimal expected-time solution. For a guaranteed worst-case bound, swap `unordered_map` for `std::map` to get O(n log n).

**Time Complexity:** O(n) average (O(n^2) worst case with unordered_map collisions)
**Space Complexity:** O(n)

---

## Feedback Given

**Problem Understanding & Clarification — 4.5/5**
Asked for constraints up front (correct), and registered the negative values — which steered him away from the sliding-window trap. Half-point off for not stating the negatives observation out loud.

**Approach & Thought Process — 5/5**
Went straight to prefix-sum + count map (structure-exploiting, not generic). Correct explanation of map init {0:1}, lookup-before-insert ordering, and the k=0 reasoning. Strong — did not default to a generic pattern.

**Code Quality & Correctness — 5/5**
Correct first pass, clean, well-commented, lookup precedes insert. No bugs.

**Complexity Analysis — 3.5/5**
Stated O(n)/O(n) (right average case) but initially asserted a flat "O(n)" and needed prompting to surface the unordered_map O(n^2) worst case and the std::map O(n log n) alternative. Recurring pattern: the true per-operation factor gets glossed. Should volunteer "O(n) average, O(n^2) worst case" unasked.

**Communication — 4/5**
Terse but accurate; dry-run was a genuine trace (good). Main gap: volunteered zero edge cases on his own (single-element, all-zeros with k=0, full-array subarray).

**Overall:** Strong round. Right structure, correct code, good reasoning. The two things holding back a perfect score are proactivity habits (volunteer complexity nuance and edge cases unasked), not knowledge gaps.

**Time Taken: 7 minutes**
