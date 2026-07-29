# DSA Round Transcript
**Date:** 2026-06-16
**Start Time:** 17:36
**End Time:** 18:18
**Duration:** 42 minutes
**Problem:** Shortest Subarray with Sum at Least K
**Topic:** Prefix Sums + Monotonic Deque
**Difficulty:** Hard

---

## Problem Statement
Given an integer array `nums` and an integer `k`, return the length of the shortest, non-empty, contiguous subarray of `nums` with a sum of at least `k`. If there is no such subarray, return `-1`.

Constraints:
- `1 <= nums.length <= 10^5`
- `-10^5 <= nums[i] <= 10^5` (values can be negative)
- `1 <= k <= 10^9`

Examples:
- `nums = [1], k = 1` → `1`
- `nums = [1, 2], k = 4` → `-1`
- `nums = [2, -1, 2], k = 3` → `3`

---

## Conversation Log

**Interviewer:** Presented the problem and asked for clarifying questions and an approach before coding.

**Aayush:** Asked about the constraints on array size and the value ranges.

**Interviewer:** Gave constraints; emphasized values can be negative, n up to 1e5, k up to 1e9 (sum can overflow 32-bit).

**Aayush:** Proposed the classic sliding window: two pointers l, r both at 0; grow r when window sum < k, shrink l when sum >= k, tracking the minimum valid window length.

**Interviewer:** Asked him to stress-test it on `nums = [-2, 5], k = 5` before committing.

**Aayush:** Recognized his algorithm returns no answer (−1) but the true answer is 1 (`[5]`).

**Interviewer:** Asked *why* it breaks — what property does sliding window rely on.

**Aayush:** "The numbers are not all positive, and sliding window relies on the fact that to increase the sum we add more elements — but with negatives that's not always true, so the approach breaks down."

**Interviewer:** Confirmed — sum is no longer monotonic in window size. Steered toward prefix sums; asked him to rewrite the condition in terms of `P` and say what he's searching for at a fixed right endpoint.

**Aayush:** "If we fix r, we need to find the maximal l such that `prefix[r] - k >= prefix[l]`."

**Interviewer:** Correct. Posed two pruning questions: (1) if `l1 < l2` and `P[l1] >= P[l2]`, is `l1` ever worth keeping? (2) once a front candidate is valid for current r, keep it for future r'?

**Aayush:** Answered #1: l1 is useless since l2 is closer and gives a smaller subarray.

**Interviewer:** Tightened the reasoning — l1 is useless because whenever l1 is valid, l2 (smaller prefix) is also valid AND closer; that's why the deque stays increasing. Asked #2.

**Aayush:** Asked for more examples.

**Interviewer:** Walked through a concrete prefix-sum example showing that once a front candidate is valid, any future r' only yields a longer subarray → pop it from the front and discard.

**Aayush:** Asked why the deque must be kept increasing.

**Interviewer:** Explained both ends — front holds the smallest prefix (enables validity early-stop), back-pop on `P[back] >= P[i]` maintains the invariant by removing dominated candidates. Asked him to code it.

**Aayush:** Wrote the full prefix-sum + monotonic-deque solution (correct, optimal). Stated O(n) time, O(n) space.

**Interviewer:** Confirmed correct and optimal. Asked him to dry-run Example 3 to verify (reinforcing self-verification).

**Aayush:** Traced `prefix = [0,2,1,3]`, gave deque states, arrived at `ans = 3`.

**Interviewer:** Flagged that the answer was right but the deque trace at i=3 was wrong — he kept `0` in the deque when his own `pop_front` had consumed it after the match. Traced intent rather than actual code; correct answer masked a wrong mental model.

**Aayush:** Noted end time.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <vector>
#include <deque>
#include <climits>
using namespace std;

class Solution {
public:
    int shortestSubarray(vector<int>& nums, int k) {
        int n = nums.size();

        vector<long long> prefix(n + 1, 0);
        for (int i = 0; i < n; i++) {
            prefix[i + 1] = prefix[i] + nums[i];
        }

        deque<int> dq;
        int ans = INT_MAX;

        for (int i = 0; i <= n; i++) {

            // Find valid subarrays
            while (!dq.empty() &&
                   prefix[i] - prefix[dq.front()] >= k) {
                ans = min(ans, i - dq.front());
                dq.pop_front();
            }

            // Maintain increasing prefix sums
            while (!dq.empty() &&
                   prefix[dq.back()] >= prefix[i]) {
                dq.pop_back();
            }

            dq.push_back(i);
        }

        return ans == INT_MAX ? -1 : ans;
    }
};
```
**Optimal Solution (if different):** Same — this is the optimal approach.

**Time Complexity:** O(n) — each index pushed and popped at most once (amortized linear).
**Space Complexity:** O(n) — prefix array + deque.

---

## Feedback Given

### Problem Understanding & Clarification — 4.5/5
Strong open. Immediately asked about array size and value ranges — and the negative-value answer is the entire crux. Asked the question that separates a correct solution from a broken sliding window, without being told negatives mattered.

### Approach & Thought Process — 4/5
Reached for the classic sliding window first (the trap), but on being asked to stress-test it, diagnosed the failure cleanly and correctly stated the underlying reason (sum not monotonic in window size with negatives → grow/shrink invariant breaks). Reformulated with prefix sums and got the right search target on his own. Leaned on worked examples to internalize the front-pop rule rather than reasoning it out.

### Code Quality & Correctness — 5/5
Clean, correct, optimal on the first write. Order of operations exactly right (front-pop matches → back-pop to maintain invariant → push), `long long` prefix to avoid overflow, correct sentinel. No bugs.

### Complexity Analysis — 4.5/5
Correct: O(n) amortized time, O(n) space. Stated plainly without hedging.

### Communication — 3.5/5
Declared the code done without tracing it; when he finally dry-ran, the deque trace was wrong (carried `0` into the final state when his own `pop_front` had consumed it). The answer matched by luck, masking a wrong mental model. Traced intent rather than actual code.

**Overall: ~4.3/5 — strong hard-problem performance.** Nailed the hardest parts (negatives trap, non-monotonicity reasoning, optimal code first try). Recurring soft spots: defaulting to the generic pattern before the structure-exploiting one, and tracing intent instead of actual code.

**Time Taken: 42 minutes**
