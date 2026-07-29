# DSA Round Transcript
**Date:** 2026-05-27
**Start Time:** 10:25
**End Time:** 11:07
**Duration:** 42 minutes
**Problem:** Smallest Range Covering Elements from K Lists
**Topic:** Heap / Ordered Set / K-Pointers
**Difficulty:** Hard

---

## Problem Statement
You have `k` lists of sorted integers in non-decreasing order. Find the smallest range `[a, b]` that includes at least one number from each of the `k` lists.

A range `[a, b]` is smaller than `[c, d]` if:
- `b - a < d - c`, or
- `b - a == d - c` and `a < c`

**Example 1:**
```
Input: nums = [[4,10,15,24,26], [0,9,12,20], [5,18,22,30]]
Output: [20, 24]
```

**Example 2:**
```
Input: nums = [[1,2,3], [1,2,3], [1,2,3]]
Output: [1, 1]
```

Constraints: `1 <= k <= 3500`, `1 <= nums[i].length <= 50`, `-10^5 <= nums[i][j] <= 10^5`.

---

## Conversation Log

**Interviewer:** Presented problem. Asked Aayush to note start time.
**Aayush:** 10:25
**Aayush:** can there be duplicates?
**Interviewer:** Yes — within a list (non-decreasing) and across lists.
**Aayush:** The way I'm thinking is k pointers, one on an element of each array. The range is min to max of pointed elements. To reduce, advance the pointer at the lowest element — remove it and insert the next element from that array.
**Interviewer:** Right intuition. What data structure for efficient min/max? When do you stop?
**Aayush:** Ordered set — set.begin() for lowest, set.rbegin() for highest. Stop when size < k.
**Interviewer:** What about duplicates (set deduplicates)?
**Aayush:** Store {element, array_index, index_in_array} — composite key always unique.
**Interviewer:** Code it up.
**Aayush:** [first version of code submitted]
**Interviewer:** Trace edge cases: (1) lowest pointer is on the last element of its list; (2) k=1.
**Aayush:** (1) Next element won't be present so nothing added. (2) Outputs the first element as l and r.
**Interviewer:** Re-check #1: with `nums[lowestArray] = [4,10,15]` and idx=2, your check `2 < 3` is true, then you access `nums[lowestArray][3]` — what's that? And for k=1, after erase the set is empty — what does `*set.rbegin()` give you?
**Aayush:** [submitted fixed code — moved range update before erase, changed bounds to `< size() - 1`]
**Interviewer:** Both bugs fixed. Now complexity?
**Aayush:** Time O(|nums[i]| * K * logK) — iterate all elements once, logK per set op. Space O(k).
**Interviewer:** Correct — O(N log k). Can you optimize the constant — simpler structure than ordered set?
**Aayush:** Min-heap and track max separately, update max only when pushing.
**Interviewer:** Exactly. Note end time.
**Aayush:** 11:07

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<vector<int>> nums{{4,4,4},{0,0,20},{5,18,22}};
    int k = nums.size();
    set<vector<int>> set;

    for(int i=0;i<k;i++) set.insert({nums[i][0], i, 0});

    int l = (*set.begin())[0];
    int r = (*set.rbegin())[0];

    while(set.size() == k) {
        auto lowestElementVec = (*set.begin());
        int lowestElement = lowestElementVec[0];
        int lowestArray = lowestElementVec[1];
        int lowestArrayIdx = lowestElementVec[2];

        auto highestElementVec = (*set.rbegin());
        int highestElement = highestElementVec[0];

        if((highestElement - lowestElement < r-l) ||
           (highestElement - lowestElement == r-l && lowestElement < l)) {
            l = lowestElement;
            r = highestElement;
        }
        set.erase(set.begin());
        if(lowestArrayIdx < (int)nums[lowestArray].size() - 1) {
            set.insert({nums[lowestArray][lowestArrayIdx+1], lowestArray, lowestArrayIdx + 1});
        }
    }
    cout<<"{"<<l<<","<<r<<"}";
    return 0;
}
```

**Optimal Solution (heap variant):**
```cpp
// min-heap of {val, listIdx, idxInList}, track curMax separately
priority_queue<tuple<int,int,int>, vector<tuple<int,int,int>>, greater<>> pq;
int curMax = INT_MIN;
for (int i = 0; i < k; i++) {
    pq.push({nums[i][0], i, 0});
    curMax = max(curMax, nums[i][0]);
}
int bestL = pq.top().__get<0>(), bestR = curMax;
while (true) {
    auto [v, listIdx, idx] = pq.top(); pq.pop();
    if (curMax - v < bestR - bestL) { bestL = v; bestR = curMax; }
    if (idx + 1 == nums[listIdx].size()) break;
    int nv = nums[listIdx][idx+1];
    curMax = max(curMax, nv);
    pq.push({nv, listIdx, idx+1});
}
```

**Time Complexity:** O(N log k) where N = total elements
**Space Complexity:** O(k)

---

## Feedback Given

### Scores
| Category | Score |
|---|---|
| Problem Understanding & Clarification | 4/5 |
| Approach & Thought Process | 5/5 |
| Code Quality & Correctness | 3/5 |
| Complexity Analysis | 4/5 |
| Communication | 4/5 |
| **Total** | **20/25** |

**Time Taken: 42 minutes**

**Highlights:**
- Strong approach: jumped to structure-exploiting k-pointer + ordered structure idea immediately, with correct greedy invariant.
- Handled duplicates cleanly by switching to composite key once prompted.
- Got optimization (heap + tracked max) immediately.

**Growth areas:**
- Recurring edge-case tracing miss: off-by-one in bounds (`idx < size()` vs `idx+1 < size()`) and `rbegin()` on empty set for k=1. Both surfaced only when asked.
- When asked to *trace* a boundary case, gave a conceptual answer instead of substituting the concrete index — the "defend before trace" pattern.
- Constraint value misremembered (|nums[i]| <= 10 vs actual 50). Minor.
- Variable shadowing `std::set` — works but avoid in interviews.
