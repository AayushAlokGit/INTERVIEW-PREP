# DSA Round Transcript
**Date:** 2026-07-05
**Start Time:** 10:50
**End Time:** 12:03
**Duration:** 73 minutes
**Problem:** Best Time to Buy and Sell Stock IV
**Topic:** Dynamic Programming
**Difficulty:** Hard

---

## Problem Statement
You are given an integer array `prices` where `prices[i]` is the price of a given stock on the `i`-th day, and an integer `k`.

Find the maximum profit you can achieve. You may complete at most `k` transactions.

Note: You may not engage in multiple transactions simultaneously — you must sell the stock before you buy again. One transaction = one buy followed by one later sell.

**Example 1**
```
Input:  k = 2, prices = [2, 4, 1]
Output: 2
Explanation: Buy on day 0 (price = 2), sell on day 1 (price = 4), profit = 2.
```

**Example 2**
```
Input:  k = 2, prices = [3, 2, 6, 5, 0, 3]
Output: 7
Explanation: Buy@2 sell@6 (profit 4), then buy@0 sell@3 (profit 3). Total = 7.
```

**Constraints:**
- `0 <= k <= 100`
- `0 <= prices.length <= 1000`
- `0 <= prices[i] <= 1000`

---

## Conversation Log

**Interviewer:** Presented the problem, asked for clarifying questions before approach.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints (k 0..100, prices length 0..1000, prices 0..1000). Noted prices can be empty, k can be 0, prices non-negative.

**Aayush:** If k is 0 and prices is an empty array then no profit possible.

**Interviewer:** Correct — k==0 or empty/length-1 prices → 0. Asked him to walk through the approach.

**Aayush:** Described a greedy: pair each price at index i with the max price[j] where j>i and price[j] >= price[i], starting from the lowest buy price, mark both indices consumed, keep count of pairings, stop when count > k or no more pairs. Proposed an ordered hashmap keyed by (price, index) with a maxHeap of candidate sell prices per key, built in O(n^2 log n).

**Interviewer:** Asked him to trace the algorithm on `prices = [1, 5, 3, 6], k = 2` — which pair is picked first, what gets consumed, final output.

**Aayush:** Traced it:
- (1,0) -> heap {(6,3),(5,1),(3,2)}
- (3,2) -> {(6,3)}
- (5,1) -> {(6,3)}
- (6,3) -> {}
- (1,0) pairs with (6,3): visited {0,3}, profit=5, cnt=1
- (3,2): no pairing (index 3 taken) -> visited {0,2,3}
- (5,1): no pairing -> visited {0,1,2,3}
- Answer = 5 with 1 transaction.

**Interviewer:** Asked if a better answer exists by hand with k=2.

**Aayush:** Yes — (1,5) and (3,6), so my algorithm is wrong.

**Interviewer:** Explained the greedy fails because the fat transaction (1→6) straddles [5,3] and blocks two better non-overlapping trades; transactions must be non-overlapping in time. Asked what technique handles interacting decisions, and what states to track.

**Aayush:** At each index, 3 options: (1) start a new transaction if none ongoing and count < k, (2) end a transaction if holding and price[i] >= buy price, (3) do nothing. Proposed `solve(prices, curProfit, i, count, lastBuyPrice)` with lastBuyPrice = -1 if idle, and the max-of-three recurrence.

**Interviewer:** Confirmed recurrence essentially correct. Asked about memoization state and complexity.

**Aayush:** Memoize on (i, count, lastBuyPrice); O(n * k * maxPrice).

**Interviewer:** Pushed: lastBuyPrice is bounded by n distinct values, and asked whether the buy price is needed as a dimension at all — could it be folded into running profit?

**Aayush:** We can simplify to whether we're currently holding a stock or not; end a transaction only if holding; the exact last buy price doesn't matter because the max handles it.

**Interviewer:** Asked him to write the full solution with (i, count, holding) state and give complexity.

**Aayush:** Wrote C++ solution accumulating `moneyAvlbl`, base case returns `moneyAvlbl`, memo key = `to_string(i)+to_string(count)+to_string(holdingStock)`. Stated TC O(nk), SC O(nk)+O(n) stack.

**Interviewer:** Pointed out base case returns moneyAvlbl but key omits it — asked whether the cached value is a function of only (i,count,holding), and whether the same state can be reached with different moneyAvlbl.

**Aayush:** Yes, moneyAvlbl also needs to be part of the key.

**Interviewer:** Noted adding moneyAvlbl to the key blows up the state space. Suggested returning incremental profit from index i onward instead of absolute money.

**Aayush:** Base case returns 0, moneyAvlbl not needed. Rewrote: buy branch = solve(...) - prices[i], sell branch = solve(...) + prices[i], memo on (i, count, holding).

**Interviewer:** Confirmed correct. Then flagged the memo key concatenation has no separator — asked if (i=1,count=11,holding=0) and (i=11,count=1,holding=0) collide.

**Aayush:** Add a separator like `=` or `|` to prevent different states mapping to the same key.

**Interviewer:** Asked for precise complexity given std::map, and how to reach clean O(nk).

**Aayush:** std::map is log(nk) per op; string building is constant (bounded lengths).

**Interviewer:** Confirmed true TC = O(n·k·log(nk)). Asked what structure drops the log factor.

**Aayush:** unordered_map works but worst case degrades; ordered map is bounded log n.

**Interviewer:** Pointed out i/count/holding are all small bounded integers — a 3D array `dp[n][k+1][2]` gives guaranteed O(1) access, clean O(nk) time/space. Wrapped up.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int solve(vector<int> &prices, int i, int count, int k, int holdingStock, map<string,int> &dp)
{
    if(i==prices.size()) return 0;

    string key = to_string(i) + to_string(count) + to_string(holdingStock); // needs a separator

    if(dp.find(key) != dp.end()) return dp[key];

    // buy stock at index i
    int buy = INT_MIN;
    if(!holdingStock && count < k) buy = max(buy, solve(prices,i+1, count+1, k, 1, dp) - prices[i]);

    // sell stock at index i
    int sell = INT_MIN;
    if(holdingStock) sell = max(sell, solve(prices,i+1,count,k,0, dp) + prices[i]);

    // do nothing at index i
    int doNothing = solve(prices,i+1,count,k,holdingStock, dp);
    return dp[key] = max(buy, max(sell, doNothing));
}
```

**Optimal Solution (clean O(nk), array-indexed memo):**
```cpp
class Solution {
public:
    vector<vector<array<int,2>>> dp;
    vector<int> prices;
    int K;

    int solve(int i, int count, int holding) {
        if (i == (int)prices.size()) return 0;
        if (dp[i][count][holding] != INT_MIN) return dp[i][count][holding];

        int best = solve(i + 1, count, holding); // do nothing
        if (!holding && count < K)
            best = max(best, solve(i + 1, count + 1, 1) - prices[i]); // buy
        else if (holding)
            best = max(best, solve(i + 1, count, 0) + prices[i]);     // sell

        return dp[i][count][holding] = best;
    }

    int maxProfit(int k, vector<int>& p) {
        prices = p; K = k;
        if (k == 0 || p.empty()) return 0;
        dp.assign(p.size(), vector<array<int,2>>(k + 1, {INT_MIN, INT_MIN}));
        return solve(0, 0, 0);
    }
};
```
**Time Complexity:** O(n·k) with array memo (his map version: O(n·k·log(nk))).
**Space Complexity:** O(n·k) for dp + O(n) recursion stack.

---

## Feedback Given

### Summary
Landed on the correct, efficient DP — O(nk) time/space with (i, count, holding) state — but via a long road. Started with an incorrect greedy and shipped two correctness bugs (memo key missing moneyAvlbl, then key-collision from no separator) that only surfaced under prompting. Recovery and state modeling were strong once redirected.

### What went well
- Recovered well from the greedy — traced [1,5,3,6], saw exactly why it fails, pivoted to DP cleanly.
- Strong state modeling — three-option recursion correct first try; nailed the lastBuyPrice → holding collapse when nudged.
- Good instincts on the incremental-profit fix once he saw the memo was caching a path-dependent value.

### What to work on
- Led with a greedy without stress-testing it; over-invested in an O(n^2 log n) heap/ordered-map machine for an approach a 4-element case breaks. "At most k transactions" + non-overlap should reflexively flag greedy as unsafe.
- Both bugs were self-verification failures — a 60-second dry-run catches both. Declared code "done" twice; interviewer had to prompt the trace. Recurring theme.
- Complexity precision — said O(nk) while using std::map; the log(nk) factor was hiding in plain sight.

### Scoring (out of 5)
| Dimension | Score | Notes |
|---|---|---|
| Problem understanding & clarification | 4 | Asked about constraints; reasoned k=0/empty base cases himself. |
| Approach & thought process | 2.5 | Started on broken greedy, over-invested; strong recovery to correct DP. |
| Code quality & correctness | 2.5 | Two real bugs shipped as "done"; final code correct after prompting. |
| Complexity analysis | 3 | Correct state space; missed the map log factor, initially mis-stated O(nk). |
| Communication | 4 | Clear, traceable reasoning; drove fixes himself under hints. |

**Overall: ~3.2 / 5** — solid problem-solving and recovery, held back by leading with an unverified greedy and shipping code without self-checking.
