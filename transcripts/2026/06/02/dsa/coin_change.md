# DSA Round Transcript
**Date:** 2026-06-02
**Start Time:** 11:41
**End Time:** 12:00
**Duration:** 19 minutes (includes a restart; Coin Change solve ~12 min)
**Problem:** Coin Change
**Topic:** Dynamic Programming (Unbounded Knapsack)
**Difficulty:** Medium

---

## Problem Statement
Given an integer array `coins` representing coin denominations and an integer `amount`, return the fewest number of coins needed to make up `amount`. If it cannot be made from any combination, return `-1`. Unlimited supply of each coin.

Examples:
- coins = [1, 2, 5], amount = 11 -> 3  (5 + 5 + 1)
- coins = [2], amount = 3 -> -1
- coins = [1], amount = 0 -> 0

Constraints:
- 1 <= coins.length <= 12
- 1 <= coins[i] <= 2^31 - 1
- 0 <= amount <= 10^4

Note: The round originally opened with "Subarray Sum Equals K," but Aayush had already solved it in a Serval round on 2026-05-31 (the new-problem check only scanned `dsa/` folders, missing `serval/`). His pasted solution to that problem was correct; the round was reset to Coin Change as a genuinely new problem, keeping the same start time.

---

## Conversation Log

**Interviewer:** Note your start time. Problem: Subarray Sum Equals K — count contiguous subarrays summing to k. Any clarifying questions?

**Aayush:** 11:41.

**Interviewer:** Start recorded. Any clarifying questions, then walk me through approach.

**Aayush:** What are the constraints?

**Interviewer:** nums.length up to 2*10^4; nums[i] in [-1000,1000]; k in [-10^7,10^7].

**Aayush:** Prefix sum + hashmap of prefix sum to count. Iterate; at each index ans += mp[prefix[i]-k].

**Interviewer:** Right approach (negatives rule out sliding window). What do you init the map with, and does update-vs-insert order matter?

**Aayush:** mp[0]=1 for empty subarray. Update answer first, then insert current prefix sum — covers k=0 with no prior prefix.

**Interviewer:** Correct on both. Write the full code.

**Aayush:** (pasted correct prefix-sum + hashmap C++ using long long, cnt[0]=1, update-then-insert)

**Interviewer:** Code is correct — but this is a problem you already solved in a Serval round on 2026-05-31 (variable-for-variable identical). My new-problem check missed the serval folder. Swapping in a fresh problem, same start time.

**Interviewer:** New problem — Coin Change. Fewest coins to make amount, -1 if impossible, unlimited supply. Clarifying questions, then approach.

**Aayush:** What are the constraints?

**Interviewer:** coins.length up to 12; coins[i] up to 2^31-1; amount up to 10^4.

**Aayush:** DP. dp[i] = min coins to make amount i. dp[coins[i]]=1; else INT_MAX. dp[0]=1. Iterate coins; dp[i]=min(dp[i], dp[i-coins[j]]+1) if i>=coins[j]. Answer dp[amount] if < INT_MAX else -1.

**Interviewer:** Recurrence is right. But (1) you wrote dp[0]=1 — what should amount=0 return, and what does dp[0]=1 give? (2) What if dp[i-coins[j]] is INT_MAX when you do +1?

**Aayush:** dp[0]=0; and handle the case when dp[i-coins[j]] == INT_MAX.

**Interviewer:** Both correct. Write the full code.

**Aayush:** (wrote correct C++ — dp sized amount+1 init INT_MAX, dp[0]=0, nested loops with guards `i>=coin && dp[i-coin]!=INT_MAX`, final ternary for -1). TC O(n), SC O(n).

**Interviewer:** When you say O(n), what is n? Two inputs matter — give the precise time complexity in both amount and coins, and name the space variable.

**Aayush:** amount*|coins|, but coins capped at 12 so amount dominates.

**Interviewer:** Exactly — O(amount*|coins|) ~ O(amount); SC O(amount). What edge cases would you test (more than one)?

**Aayush:** amount=0; odd amount with only even coins; even amount with odd coins.

**Interviewer:** Trace coins=[2], amount=3.

**Aayush:** dp[0]=0, dp[2]=1, rest INT_MAX.

**Interviewer:** Correct — dp[3] needs dp[1] which is INT_MAX, guard skips it, returns -1. Wrapped up; asked for end time.

**Aayush:** 12:00.

**Interviewer:** Delivered feedback.

---

## Solution

**Aayush's Final Solution:**
```cpp
int coinChange(vector<int>& coins, int amount) {
    vector<int> dp(amount + 1, INT_MAX);
    dp[0] = 0;
    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (i >= coin && dp[i - coin] != INT_MAX) {
                dp[i] = min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    return (dp[amount] == INT_MAX) ? -1 : dp[amount];
}
```

**Optimal Solution (if different):** Same — this is the optimal bottom-up DP.

**Time Complexity:** O(amount * |coins|), effectively O(amount) since |coins| <= 12.
**Space Complexity:** O(amount).

---

## Feedback Given

# Feedback — Coin Change

## Problem Understanding & Clarification — 9/10
Asked for constraints first, both times, before committing — directly counters his most persistent gap (skipping clarifications). It paid off (spotted negatives in Subarray-Sum, small |coins| here). Minor: could have asked what to return for the impossible case rather than relying on the spec.

## Approach & Thought Process — 8/10
Went straight to the correct unbounded-knapsack DP with the right recurrence and clear state definition. One slip: stated dp[0]=1 in init (should be 0); corrected immediately on probing. Lock down base-case precision.

## Code Quality & Correctness — 9/10
Clean and correct. Proactively included both guards (i>=coin and dp[i-coin]!=INT_MAX) in code, closing the overflow hole. Correct sizing and final ternary.

## Complexity Analysis — 7/10
Initially said "O(n)" without defining n (recurring imprecision). When pushed, landed cleanly on O(amount*|coins|) and argued |coins|<=12 makes amount dominate, plus O(amount) space. Give the precise multi-factor bound first, unprompted.

## Communication — 8/10
Better on edge cases — volunteered three (amount=0, odd/even, even/odd) rather than one, and traced [2],amount=3 correctly instead of asserting. Direct progress on shallow-edge-coverage and defend-instead-of-trace weaknesses.

### Net
Strong, efficient solve — correct optimal approach reached fast, improved clarification and edge-case habits. Tighten: (1) base-case precision (dp[0]); (2) state precise multi-factor complexity on first pass instead of vague "O(n)."
