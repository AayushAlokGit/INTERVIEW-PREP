# DSA Round Transcript
**Date:** 2026-05-07
**Start Time:** 10:01 PM
**End Time:** 10:13 PM
**Duration:** 12 minutes
**Problem:** Largest Rectangle in Histogram
**Topic:** Monotonic Stack
**Difficulty:** Hard

---

## Problem Statement
Given an array of integers `heights` representing the histogram's bar heights where the width of each bar is 1, return the area of the largest rectangle in the histogram.

**Example 1:** heights = [2,1,5,6,2,3] → Output: 10 (bars of heights 5 and 6, area = 5 * 2 = 10)
**Example 2:** heights = [2,4] → Output: 4

**Constraints:** 1 <= heights.length <= 10^5, 0 <= heights[i] <= 10^4

---

## Conversation Log

**Interviewer:** Asked for start time and clarifying questions.

**Aayush:** 10:01

**Aayush:** What if there are no elements in heights?

**Interviewer:** Per constraints, length >= 1, but good instinct.

**Aayush:** If each bar is of height 0?

**Interviewer:** Answer is 0; code should handle this naturally.

**Aayush:** At each index the rectangle can keep expanding to left until it encounters a smaller height, similarly for the right. So in 2 passes through the array we can calculate left[i] and right[i] where left[i] = index of element < height[i] and to its left (-1 if none), similarly for right[i]. Then area = height[i] * (right[i] - left[i] - 1). Answer = max.

**Interviewer:** Good framing. How do you compute left[i] and right[i] efficiently?

**Aayush:** Use a monotonic stack to calculate left and right arrays. For left iterate left to right, for right iterate right to left. At index i keep popping elements from stack which are >= height[i].

**Interviewer:** Exactly — O(n) total. Code it up.

**Aayush:** (Wrote initial C++ — see below. Bug: right[i] = -1 when stack empty, should be n.)

**Interviewer:** Bug spotted: right[i] should be n, not -1, when stack empty. Trace on [1,2,3,4,5] — correct answer 9, your code gives 5.

**Aayush:** Time complexity is O(n) and space complexity is O(n).

**Interviewer:** Hold on — please fix the bug first and re-verify on [1,2,3,4,5].

**Aayush:** (Fixed right[i] sentinel to n.)

**Interviewer:** Verified. Bonus: can you do single pass?

**Aayush:** We fix the right boundary at i and left boundary should be st.top() if st is not empty and st.top() >= height[i], then calculate and update area and push height[i] to stack. However if st.top() < height[i] we pop.

**Interviewer:** Description was inverted — standard approach maintains an increasing stack of indices and finalizes the popped bar's rectangle when a smaller element appears. Skipped coding this.

**Interviewer:** Asked for end time.

**Aayush:** 10:13

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> heights{2,1,5,6,2,3};
    int n = heights.size();
    stack<int> st;
    vector<int> left(n), right(n);
    left[0] = -1;
    st.push(0);
    for (int i = 1; i < n; i++) {
        while (!st.empty() && heights[st.top()] >= heights[i]) st.pop();
        left[i] = st.empty() ? -1 : st.top();
        st.push(i);
    }
    while (!st.empty()) st.pop();
    right[n-1] = n;
    st.push(n-1);
    for (int i = n-2; i >= 0; i--) {
        while (!st.empty() && heights[st.top()] >= heights[i]) st.pop();
        right[i] = st.empty() ? n : st.top();
        st.push(i);
    }
    int ans = 0;
    for (int i = 0; i < n; i++)
        ans = max(ans, heights[i] * (right[i] - left[i] - 1));
    cout << "ans is " << ans << endl;
    return 0;
}
```

**Optimal Solution (single pass):**
```cpp
int largestRectangleArea(vector<int>& heights) {
    stack<int> st;
    int n = heights.size(), ans = 0;
    for (int i = 0; i <= n; i++) {
        int h = (i == n) ? 0 : heights[i];
        while (!st.empty() && heights[st.top()] >= h) {
            int top = st.top(); st.pop();
            int width = st.empty() ? i : i - st.top() - 1;
            ans = max(ans, heights[top] * width);
        }
        st.push(i);
    }
    return ans;
}
```

**Time Complexity:** O(n)
**Space Complexity:** O(n)

---

## Feedback Given

### Scoring Rubric

| Category | Score (/5) | Notes |
|---|---|---|
| Problem Understanding & Clarification | 4 | Asked two clarifying questions — improvement! |
| Approach & Thought Process | 4.5 | Immediately framed prev/next smaller via monotonic stack. |
| Code Quality & Correctness | 3 | Bug in right[i] sentinel — used -1 instead of n when stack empty. Manifests on increasing arrays. |
| Complexity Analysis | 5 | O(n) time and space, correct and confident. |
| Communication | 3.5 | Initially skipped over bug feedback and jumped to complexity. Single-pass description was inverted. |

**Overall: 20 / 25**

### Strengths
- Clarifying questions (improvement)
- Right pattern recognition immediately (monotonic stack)
- Clean two-pass implementation
- Correct complexity analysis

### Areas to Improve
1. Sentinel values must match the area formula. Right → n, left → -1.
2. Absorb feedback before pivoting — when bug is flagged, fix and re-verify before moving on.
3. Practice single-pass monotonic stack — same O(n), cleaner code.
