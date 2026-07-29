# DSA Round Transcript
**Date:** 2026-05-07
**Start Time:** 17:01
**End Time:** 17:39
**Duration:** 38 minutes
**Problem:** Sum of Subarray Minimums (LeetCode 907)
**Topic:** Monotonic Stack / Contribution Counting
**Difficulty:** Medium

---

## Problem Statement

Given an array of integers `arr`, find the sum of `min(b)`, where `b` ranges over every (contiguous) subarray of `arr`.

Since the answer may be large, return the answer modulo `10^9 + 7`.

**Example 1:**
```
Input: arr = [3,1,2,4]
Output: 17
```

**Example 2:**
```
Input: arr = [11,81,94,43,3]
Output: 444
```

**Constraints:**
- `1 <= arr.length <= 3 * 10^4`
- `1 <= arr[i] <= 3 * 10^4`

---

## Conversation Log

**Interviewer:** [Initial problem: Largest Rectangle in Histogram]

**Aayush:** solved this

**Interviewer:** [Switched to Sum of Subarray Minimums]

**Aayush:** 17:01 (start time)

**Interviewer:** Go ahead — clarifying questions and approach?

**Aayush:** Brute force would be generating all subarrays in O(n*n) and then checking the min O(n) so total O(n*N*N).

**Interviewer:** Right, can shrink to O(n²). But n=3*10^4 makes that ~10^9. Hint: think contribution per element.

**Aayush:** Right the contribution for each element would be only for the subarrays where it would be the minimum. So if we have left[i] = index to left of i and > nums[i], right[i] = index to right of i and > nums[i], the number of arrays where nums[i] is minimum is (i-left[i])*(right[i]-i)-1.

**Interviewer:** Probed: traced [3,1,2,4] at i=1, manual count = 6, his formula = 5. The -1 is wrong. Also asked about duplicates [1,1].

**Aayush:** Okay to handle duplicates left and right should point to first index >= nums[i]. And the formula is i-left[i] + right[i] - i - 1.

**Interviewer:** Two issues — both sides >= undercounts (shows trace: 1+1=2 vs actual 3). Formula should be multiplication, not addition.

**Aayush:** The formula is (i-left[i])*(right[i]-i).

**Interviewer:** Good. But duplicate fix still wrong — both >= undercounts. Need asymmetric strictness.

**Aayush:** We could keep a map from value to all indices it appears in. For contribution, we counted extra subarrays. So we remove duplicate counted subarrays.

**Interviewer:** Explained: with both >=, undercounts (not overcounts). With both strict >, overcounts. Cleanest fix: asymmetric strictness — one side strict <, other side <=. Showed trace for [1,1] proving total = 3.

**Aayush:** [Submitted code with left using >= (strict <), right using > (non-strict <=)]

**Interviewer:** Trace confirmed correct on [3,1,2,4] = 17. But: missed modulo, fragile variable names left/right, integer overflow risk.

**Aayush:** [Resubmitted with mod applied per-factor]

**Interviewer:** Works due to long long promotion via mod variable, but convoluted. Idiomatic fix is `1LL * a * b * c`. Also second bug: ans not modded after final addition.

**Aayush:** [Final fix with `ans %= mod` after each iteration]

**Aayush:** 17:39 (end time)

---

## Solution

**Aayush's Final Solution (C++):**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> nums{3,1,2,4};
    int n = nums.size();
    stack<int> st;
    vector<int> left(n), right(n);
    long long mod = 1e9 + 7;

    left[0] = -1;
    st.push(0);
    for (int i = 1; i < n; i++) {
        while (!st.empty() && nums[st.top()] >= nums[i]) st.pop();
        left[i] = st.empty() ? -1 : st.top();
        st.push(i);
    }

    while (!st.empty()) st.pop();

    right[n-1] = n;
    st.push(n-1);
    for (int i = n-2; i >= 0; i--) {
        while (!st.empty() && nums[st.top()] > nums[i]) st.pop();
        right[i] = st.empty() ? n : st.top();
        st.push(i);
    }

    long long ans = 0;
    for (int i = 0; i < n; i++) {
        ans += (((i-left[i]) % mod) * ((right[i]-i) % mod) * (nums[i] % mod));
        ans %= mod;
    }
    cout << ans << endl;
    return 0;
}
```

**Idiomatic Solution:**
```cpp
const long long MOD = 1e9 + 7;
long long ans = 0;
for (int i = 0; i < n; i++) {
    ans = (ans + 1LL * (i - left[i]) * (right[i] - i) * nums[i]) % MOD;
}
```

**Time Complexity:** O(n)
**Space Complexity:** O(n)

---

## Feedback Given

**Time Taken: 38 minutes**

### Scoring
| Criterion | Score |
|---|---|
| Problem Understanding & Clarification | 3/5 |
| Approach & Thought Process | 4/5 |
| Code Quality & Correctness | 3/5 |
| Complexity Analysis | 4/5 |
| Communication | 4/5 |

**Overall: 18/25 (~72%)**

### Recurring Patterns
- Missed modulo on first submission — pattern of skimming problem statement.
- Didn't trace before submitting — would have caught int overflow.
- Convoluted overflow fix instead of `1LL *` idiom.
- Didn't reach asymmetric-strictness pattern self; resorted to ad-hoc duplicate counting.

### Wins
- Recognized contribution counting immediately.
- Correct asymmetric strictness in code (>= left, > right), even without articulating the principle.
- Stated complexity proactively.

### What to Practice
- Read full problem statement before coding (modulo, value bounds).
- Asymmetric strictness pattern for monotonic stack with duplicates.
- Idiomatic `1LL * a * b * c` for overflow guard in C++.
