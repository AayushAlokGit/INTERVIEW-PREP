# DSA Round Transcript
**Date:** 2026-06-01
**Start Time:** 11:16
**End Time:** 11:38
**Duration:** 22 minutes
**Problem:** Maximum Product Subarray
**Topic:** Dynamic Programming (Kadane variant)
**Difficulty:** Medium

---

## Problem Statement
Given an integer array `nums`, find a contiguous subarray (containing at least one number) that has the largest product, and return that product. The answer is guaranteed to fit in a 32-bit integer.

**Example 1:**
```
Input:  nums = [2, 3, -2, 4]
Output: 6
Explanation: The subarray [2, 3] has the largest product 6.
```

**Example 2:**
```
Input:  nums = [-2, 0, -1]
Output: 0
Explanation: The result cannot be 2, because [-2,-1] is not contiguous.
```

**Constraints:**
- `1 <= nums.length <= 2 * 10^4`
- `-10 <= nums[i] <= 10`
- The product of any prefix or suffix of `nums` is guaranteed to fit in a 32-bit integer.

---

## Conversation Log

**Interviewer:** Presented the problem, examples, and asked for clarifying questions.

**Aayush:** What are the constraints?

**Interviewer:** Provided constraints (length up to 2*10^4, values in [-10,10], non-empty, products fit in 32-bit).

**Aayush:** Brute force: compute prefix[i] = product of nums[0..i]. Fix l and r, get product of subarray from prefix. Get max over all subarrays in O(n^2), SC O(n).

**Interviewer:** How exactly do you compute product(l..r) from the prefix array? Where does 0 break it?

**Aayush:** We need to handle 0 appropriately.

**Interviewer:** What specifically goes wrong?

**Aayush:** If there is a 0 in [l:r] the product always becomes 0, so the prefix (division) logic only works if there are no zeros in the array.

**Interviewer:** Agreed — division-via-prefix only works zero-free. Cleaner brute force: fix l, sweep r with a running product — O(n^2)/O(1). Can you do better than O(n^2)? What makes products trickier than Kadane's max-sum?

**Aayush:** At each element, choose whether the max-product subarray ending at i is just nums[i] or extends a previous subarray. If nums[i] < 0: maxProd[i] = max(nums[i], minProd[i-1]*nums[i]); if nums[i] > 0: maxProd[i] = max(nums[i], maxProd[i-1]*nums[i]). So track both min and max ending at i. Symmetrically for minProd[i].

**Interviewer:** Correct. Does the nums[i] > 0 branch also cover nums[i] == 0?

**Aayush:** For nums[i] == 0 we can treat it the same as nums[i] > 0 (both collapse to 0).

**Interviewer:** Right. Write the full code.

**Aayush:** (submitted O(n) min/max solution with swap trick — see below)

**Interviewer:** Dry-run on nums = [-2, 3, -4]. Give maxProd, minProd, ans after each iteration.

**Aayush:** i=1 -> maxProd=3, minProd=-6, ans=-1. i=2 -> maxProd=24, minProd=-12, ans=24.

**Interviewer:** maxProd/minProd correct, final 24 correct, but recheck ans at i=1. Where did -1 come from?

**Aayush:** ans starts at -2.

**Interviewer:** So ans at i=1 = max(-2, 3) = 3, not -1. A slip?

**Aayush:** Yes.

**Interviewer:** Confirmed — trace slip, not a code bug. Complexity?

**Aayush:** TC O(n), SC O(1).

**Interviewer:** Correct and optimal. Follow-ups: (1) alternative O(n) without min/max? (2) what worries you without the 32-bit guarantee?

**Aayush:** (2) Need to change int to long long for maxProd/minProd.

**Interviewer:** Good. Revealed the prefix/suffix two-pass alternative. Asked Aayush to note end time.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <vector>
#include <algorithm>
using namespace std;

class Solution {
public:
    int maxProduct(vector<int>& nums) {
        int maxProd = nums[0];
        int minProd = nums[0];
        int ans = nums[0];

        for (int i = 1; i < nums.size(); i++) {
            if (nums[i] < 0)
                swap(maxProd, minProd);

            maxProd = max(nums[i], maxProd * nums[i]);
            minProd = min(nums[i], minProd * nums[i]);

            ans = max(ans, maxProd);
        }

        return ans;
    }
};
```

**Optimal Solution (same complexity; alternative lens):**
```cpp
// Prefix/suffix two-pass: max-product subarray is a prefix or suffix
// of each zero-delimited segment (chopping one end removes one negative).
int maxProduct(vector<int>& nums) {
    int n = nums.size(), ans = INT_MIN, pre = 1, suf = 1;
    for (int i = 0; i < n; i++) {
        pre = (pre == 0 ? 1 : pre) * nums[i];
        suf = (suf == 0 ? 1 : suf) * nums[n - 1 - i];
        ans = max(ans, max(pre, suf));
    }
    return ans;
}
```
**Time Complexity:** O(n)
**Space Complexity:** O(1)

---

## Feedback Given

**Overall:** Strong round. Reached the optimal O(n)/O(1) solution cleanly and independently; code correct on first write. The one wobble was a trace slip, not a logic error.

**What went well**
- Asked for constraints upfront — good early habit.
- Brute force was honest about its flaw — caught the zero/division problem without prompting.
- Nailed the core insight: a negative flips ordering, so carry both max and min; crisp recurrence + the swap trick.
- Complexity instant and correct.
- Overflow answer (long long) correct.

**What to sharpen**
- Trace slip: confidently reported ans = -1 at i=1 when it was 3. Surrounding variable values were correct, so careless readout, not misunderstanding — but asserting wrong intermediate values can make an interviewer doubt a correct solution. Slow down on the accumulator variable when dry-running. Recurring pattern.
- Volunteer alternatives/optimizations unprompted — prefix/suffix lens and overflow concern only surfaced after prompting.

**Scoring (out of 5)**
| Criterion | Score | Note |
|---|---|---|
| Problem understanding & clarification | 4.0 | Asked constraints; could've probed value range implications |
| Approach & thought process | 4.5 | Reached optimal independently, clean reasoning |
| Code quality & correctness | 5.0 | Correct first try, elegant swap, zeros handled |
| Complexity analysis | 5.0 | Immediate and correct |
| Communication | 3.5 | Trace slip on ans; optimizations only when prompted |

**Time Taken: 22 minutes**
