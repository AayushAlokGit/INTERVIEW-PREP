# DSA Round Transcript
**Date:** 2026-06-15
**Start Time:** 9:05
**End Time:** 9:40
**Duration:** 35 minutes
**Problem:** Continuous Subarray Sum
**Topic:** Prefix Sum / Hashmap (Modular Arithmetic)
**Difficulty:** Medium

---

## Problem Statement
Given an integer array `nums` and an integer `k`, return `true` if `nums` has a **good subarray**, otherwise `false`.

A **good subarray** is a subarray where:
1. its length is **at least two**, and
2. the sum of its elements is a **multiple of `k`**.

`0` is always considered a multiple of `k`.

**Example 1:** `nums = [23, 2, 4, 6, 7], k = 6` → `true` ([2,4] sums to 6).
**Example 2:** `nums = [23, 2, 6, 4, 7], k = 6` → `true` (whole array sums to 42 = 7*6).
**Example 3:** `nums = [23, 2, 6, 4, 7], k = 13` → `false`.

**Constraints:**
```
1 <= nums.length <= 10^5
0 <= nums[i] <= 10^9
0 <= sum(nums[i]) <= 2^31 - 1
1 <= k <= 10^9
```

---

## Conversation Log

**Interviewer:** Presented the problem and asked for clarifying questions.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints; noted `nums[i]` can be 0 and `k >= 1`.

**Aayush:** Brute force — fix the right end of the subarray, sweep the left end over [0, right-1], and check if each subarray sum is divisible by k. Precompute prefix sums. TC O(n²), SC O(n) for prefix array.

**Interviewer:** Correct. Can we do better? Look at what the divisibility check looks like in terms of prefix sums.

**Aayush:** Keep a hashmap of sums encountered; at index i, check if `prefix[i] - x*k` is present, incrementing x from 1 while it's < prefix[i]. Complexity becomes O(n * sum(nums)/k).

**Interviewer:** That's pseudo-polynomial. Reframe the condition `(prefix[i] - prefix[j]) % k == 0` — what does it say about `prefix[i] % k` vs `prefix[j] % k`?

**Aayush:** `p[i]%k == p[j]%k`, so we need to track `p[i]%k` in the hashmap.

**Interviewer:** Exactly — store remainders. Two details: (1) enforce length ≥ 2, (2) what do you map each remainder to?

**Aayush:** Store indices; at each match iterate through the indices to find one satisfying length ≥ 2.

**Interviewer:** That degrades back toward O(n²). You only need one index per remainder — which one?

**Aayush:** Keep the earliest index so the left boundary is farthest; if the farthest possible still isn't ≥2, it's impossible.

**Interviewer:** Correct. Code it up.

**Aayush:** (submitted C++ solution — see below — claiming TC O(n), SC O(n) + O(k) for map)

**Interviewer:** Walk me through what it returns for `nums = [1,2,3], k = 6`.

**Aayush:** Need to pre-seed the map with `{0:-1}` to account for when the prefix itself is divisible by k.

**Interviewer:** Correct — that's the bug. Re-trace `[1,2,3], k=6` with the fix.

**Aayush:** seed `{0:-1}`; i=0 → `{0:-1, 1:0}`; i=1 → `{0:-1,1:0,3:1}`; i=2 → rem 0 found at -1, left=0 right=2. Returns true.

**Interviewer:** One more probe — trace `nums = [6,6], k=6` with your current code (including the separate `mp[prefix[0]%k]=0` seed).

**Aayush:** seed `{0:-1}`; i=0 → `{0:0}`; i=1 → `{0:0}`; left=-1 right=-1. The `prefix[0]%k` initialization is a bug — the loop should run 0..n and the map should just be seeded with `{0:-1}`.

**Interviewer:** Trace your proposed fix on `[6,6], k=6` — you still have the unconditional `mp[rem]=i` at the end. Step through i=1.

**Aayush:** `mp[curRemainder]` should be set only if the key is not found; if already present, keep the old (earliest) value.

**Interviewer:** That's the fix. Confirmed correct & optimal. Then discussed complexity refinements.

**Aayush:** (2) Reduce prefix-array space with a running-sum variable. (1) Map contains a key per remainder, 0..k-1 keys.

**Interviewer:** Tightened map space to O(min(n, k)).

---

## Solution
**Aayush's Final Solution (corrected during discussion):**
```cpp
#include <bits/stdc++.h>
using namespace std;

bool checkSubarraySum(vector<int>& nums) {
    int n = nums.size();
    int k = 6;
    unordered_map<int,int> mp;
    mp[0] = -1;            // empty-prefix base case
    int running = 0;
    for (int i = 0; i < n; i++) {
        running += nums[i];
        int rem = running % k;
        if (mp.find(rem) != mp.end()) {
            if (i - mp[rem] >= 2) return true;
        } else {
            mp[rem] = i;   // insert only if absent → keep earliest index
        }
    }
    return false;
}
```

**Bugs found via forced dry-run:**
1. Missing `{0:-1}` seed → good subarrays starting at index 0 missed.
2. Unconditional `mp[rem] = i` clobbered the earliest index → broke the length-≥2 check (e.g. `[6,6], k=6`).

**Time Complexity:** O(n)
**Space Complexity:** O(min(n, k)) with running sum

---

## Feedback Given

### Strengths
- Asked for constraints up front (improvement).
- Reached the optimal modular-arithmetic solution with only light nudges; derived `prefix[i]%k == prefix[j]%k` quickly.
- Self-corrected "store all indices" → "keep earliest index" with sound reasoning.
- Sharp complexity analysis; knew the running-sum space optimization and key-count bound.

### Areas to work on
- Declared code done with two correctness bugs (missing `{0:-1}` base case; unconditional overwrite breaking the earliest-index invariant). Neither surfaced until forced to dry-run. Recurring pattern: stops at "looks plausible" instead of tracing an adversarial small input. Habit to build: after writing, pick the nastiest small input (not the example) and trace line by line.
- First optimization (iterate x over multiples of k) was brute-force-with-a-hashmap rather than exploiting modular structure; reached the structural insight only after a nudge.

### Scoring (out of 5)
| Criterion | Score | Note |
|---|---|---|
| Problem understanding & clarification | 3.5 | Asked constraints; could probe length/zero semantics more |
| Approach & thought process | 4.0 | Reached optimal with light hints |
| Code quality & correctness | 2.5 | Two bugs, both surfaced only via forced dry-run |
| Complexity analysis | 4.5 | Precise, knew space optimization |
| Communication | 3.5 | Traces accurately when prompted; doesn't self-initiate verification |

**Time Taken: 35 minutes**
