# DSA Round Transcript
**Date:** 2026-06-17
**Start Time:** 15:32
**End Time:** 16:26
**Duration:** 54 minutes
**Problem:** Maximum Width Ramp
**Topic:** Monotonic Stack / Arrays
**Difficulty:** Medium

---

## Problem Statement
A **ramp** in an integer array `nums` is a pair `(i, j)` where `i < j` and `nums[i] <= nums[j]`. The **width** of such a ramp is `j - i`. Given an array `nums`, return the maximum width of any ramp. If there is no ramp, return `0`.

**Example 1:**
```
Input:  nums = [6, 0, 8, 2, 1, 5]
Output: 4   (ramp (1,5): 0 <= 5, width 4)
```

**Example 2:**
```
Input:  nums = [9, 8, 1, 0, 1, 9, 4, 0, 4, 1]
Output: 7   (ramp (2,9): 1 <= 1, width 7)
```

**Constraints:**
- `2 <= nums.length <= 5 * 10^4`
- `0 <= nums[i] <= 5 * 10^4`

---

## Conversation Log

**Interviewer:** Presented the problem and asked for clarifying questions.
**Aayush:** "what are the constraints?"
**Interviewer:** Gave constraints (n up to 5e4, values 0..5e4).
**Aayush:** "brute force is checking all pairs in O(n*n)"
**Interviewer:** Confirmed; noted ~2.5B ops too slow. Asked for a better approach exploiting structure.
**Aayush:** Proposed sorting a vector of (value, index) pairs ascending by value; iterate, and since all earlier elements have value <= current, track the minimum index seen so far and update answer with currentIndex - minIndex. Explained sorting fixes the value ordering so all prior indices are valid left endpoints.
**Interviewer:** Confirmed approach sound; asked how he'd handle ties (equality allowed) given the sort.
**Aayush:** "sort by values ascending and then by indices ascending"
**Interviewer:** Correct tie-break. Asked him to code it.
**Aayush:** Wrote sort-based C++ solution; stated O(n log n) time, O(n) space.
**Interviewer:** Noted his self-chosen test array was nearly sorted (weak test). Asked him to dry-run on Example 2 (has duplicates).
**Aayush:** Produced correct sorted vector: {0,3}{0,7}{1,2}{1,4}{1,9}{4,6}{4,8}{8,1}{9,0}{9,5}
**Interviewer:** Asked him to continue the trace.
**Aayush:** "minInd starts at 3->2->1->0; ans 0->4->7" — correct, final 7.
**Interviewer:** Confirmed correct trace. Asked if he could do better than O(n log n).
**Aayush:** "for a given j we need to find the furthest i such n[i]<=n[j] and i<j"
**Interviewer:** Right framing. Hinted: for two candidate left endpoints a<b, is b ever better?
**Aayush:** "a is always the better option"
**Interviewer:** Confirmed dominance; said candidates form a strictly decreasing sequence. Asked to build it and find best j in O(n).
**Aayush:** "why strictly decreasing?"
**Interviewer:** Explained equal values: earlier equal-valued index dominates later one, so push only when strictly less.
**Aayush:** "scan j from right to left and if nums[st.top()] > nums[j] then skip else update answer and continue"
**Interviewer:** Asked: do you pop after updating, and why is it safe? Why right-to-left?
**Aayush:** "we can pop it since for any future j' < j the top will only give a lesser width" and "to ensure that j > top index"
**Interviewer:** Confirmed pop reasoning; sharpened the right-to-left reason (first qualifying j is the largest → max width → safe to pop). Asked for the O(n) code.
**Aayush:** Wrote the monotonic-stack solution (build decreasing stack, scan from right, pop while qualifying).
**Interviewer:** Traced both examples — correct. Asked for complexity and why the nested while is O(n).
**Aayush:** "O(n) and O(n) space, because each index visited at most once"
**Interviewer:** Confirmed amortized O(1) per j; optimal solution reached.

---

## Solution
**Aayush's Final Solution (optimal, O(n)):**
```cpp
class Solution {
public:
    int maxWidthRamp(vector<int>& nums) {
        int n = nums.size();
        stack<int> st;
        // Build decreasing stack of candidate left endpoints
        for (int i = 0; i < n; i++) {
            if (st.empty() || nums[i] < nums[st.top()]) {
                st.push(i);
            }
        }
        int ans = 0;
        // Scan from right
        for (int j = n - 1; j >= 0; j--) {
            while (!st.empty() && nums[st.top()] <= nums[j]) {
                ans = max(ans, j - st.top());
                st.pop();
            }
        }
        return ans;
    }
};
```

**Earlier sort-based solution (O(n log n)):**
```cpp
vector<pair<int,int>> v;            // (value, index)
for (int i = 0; i < n; i++) v.push_back({nums[i], i});
sort(v.begin(), v.end());           // by value asc, then index asc
int minInd = v[0].second, ans = 0;
for (int i = 1; i < n; i++) {
    minInd = min(minInd, v[i].second);
    ans = max(ans, v[i].second - minInd);
}
```

**Time Complexity:** O(n) (stack version); O(n log n) (sort version)
**Space Complexity:** O(n)

---

## Feedback Given

**Scoring**

1. **Problem Understanding & Clarification — 4/5** — Asked about constraints (good). Didn't proactively confirm equality semantics of the ramp condition, but handled ties correctly when prompted. Slightly reactive.

2. **Approach & Thought Process — 5/5** — Standout. Brute force → sort O(n log n) → O(n) stack, each step motivated by real structure. Dominance argument articulated cleanly. When asked to optimize past O(n log n), reframed the subproblem and drove to the answer instead of stalling.

3. **Code Quality & Correctness — 5/5** — Both solutions correct on first write. Stack version textbook clean (strict-less push, right-to-left pop). No bugs.

4. **Complexity Analysis — 5/5** — Accurate throughout; correctly justified why the nested while is still O(n) (each index popped at most once / amortized).

5. **Communication — 5/5** — Clear reasoning; dry-run on Example 2 traced the actual code correctly (minInd 3->2->1->0, ans 0->4->7). Correctly justified popping and scan direction.

**Overall:** Strong round. Reached optimal O(n) solution independently with correct code, correct complexity, and a faithful dry-run. Thinnest spots: (a) clarify equality/edge semantics proactively rather than after a nudge; (b) self-chosen test was a near-sorted array that wouldn't catch a tie bug — when picking own tests, deliberately choose adversarial inputs (duplicates, all-decreasing, single valid pair).

**Time Taken: 54 minutes**
