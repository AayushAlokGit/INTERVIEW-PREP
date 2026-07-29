# DSA Round Transcript
**Date:** 2026-05-13
**Start Time:** 17:47
**End Time:** 18:54
**Duration:** 67 minutes
**Problem:** Minimum Number of Intervals to Cover Target Range
**Topic:** Greedy + Heap
**Difficulty:** Medium-Hard

---

## Problem Statement

You are given a list of intervals `intervals` where `intervals[i] = [left_i, right_i]`, and an integer `target`.

Each interval has a cost of 1. You want to cover the range `[0, target]` (fully, inclusive on both ends) using the minimum number of intervals from the list. You may use each interval at most once.

Return the minimum number of intervals needed to cover `[0, target]`, or `-1` if it is impossible.

**Constraints:**
- `1 <= intervals.length <= 10^5`
- `0 <= left_i <= right_i <= 10^9`
- `1 <= target <= 10^9`

**Example 1:**
```
Input:  intervals = [[0,4],[3,7],[6,10]], target = 10
Output: 3
```

**Example 2:**
```
Input:  intervals = [[0,6],[3,10],[0,3],[5,10]], target = 10
Output: 2
```

**Example 3:**
```
Input:  intervals = [[0,3],[5,10]], target = 10
Output: -1
```

---

## Conversation Log

**Interviewer:** Please note your current time — that'll be our round start time. Here's your problem: [presented problem above]. Do you have any clarifying questions before you start?

**Aayush:** 17:47

**Interviewer:** Round start time noted. Take your time, think aloud — what's your initial read on this problem?

**Aayush:** A recursive method where solve(i, maxPossibleDistanceAchievable, target) -> min number of intervals needed with intervals[i:n-1] and reachable distance so far = maxPossibleDistanceAchievable. The answer would be solve(0,0,target). Base cases: if maxPossibleDistance >= target return 0. If i==intervals.size() return INF. solve(i, maxDist, target) = min(solve(i+1, max(r[i],maxDist), target) + 1, solve(i+1,maxDist, target))

**Interviewer:** Good instinct to think recursively first. Two things to probe: (1) In your transition, you always allow taking interval i regardless of maxDist. What if left[i] > maxDist — can you include it without creating a gap? (2) What's the state space? Can you memoize efficiently?

**Aayush:** 1. Yes we would also need to add check that if left[i] > maxDist then we can't include this interval and should return INF. 2. The state space is indices of intervals and maxPossibleDistance which can be up to 10^9.

**Interviewer:** Exactly right on both. Memoization is off the table. You're on an exponential brute force. How do you fix this — is there a greedy choice you can make?

**Aayush:** Sort by left endpoint, maintain maxPossibleDistance initialised to 0, iterate through intervals left to right. Push right[i] of intervals which have left <= maxPossibleDistance into a maxHeap. If at index i maxDist < l[i] then we must use the interval with max achievable right side from our maxHeap. If maxHeap empty or maxPossibleDistance still < left[i] return -1. Otherwise push right[i] onto heap.

**Interviewer:** Good — that's a valid greedy approach. What happens after the loop ends if maxDist < target? And when you pop from the heap and update maxDist, do you need to re-examine the current interval i?

**Aayush:** After the loop if maxDist < target and maxH is empty || maxH.top() < target then not possible, else if maxH.top() >= target then need to include another 1 interval.

**Interviewer:** Correct — and only one pop is ever needed post-loop because the heap is a max-heap: if the best remaining interval can't reach target, nothing can. Go ahead and code it up.

**Aayush:** [submitted first version with unconditional cnt++ at the end]

**Interviewer:** Code looks clean. Trace through this case: intervals = [[0,10],[4,15]], target=10. What does your code output?

**Aayush:** [identified the bug — unconditional cnt++ fires even when maxDist >= target at loop exit — and submitted corrected version with conditional cnt++]

**Interviewer:** Correct fix. What's the time and space complexity?

**Aayush:** Time is O(n log n) — sorting + iteration + heap operations. Space is O(n).

**Interviewer:** Please note the current time for end of round.

**Aayush:** 18:54

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<vector<int>> intervals {{0,4},{3,7},{6,10}};
    int target=10;
    int n = intervals.size();
    priority_queue<int> maxH;

    sort(intervals.begin(), intervals.end());

    int cnt = 0;
    int i=0;
    int maxDist = 0;
    while(i<n && maxDist < target)
    {
        if(intervals[i][0] > maxDist)
        {
            if(maxH.empty() || maxH.top() < intervals[i][0])
            {
                cout<<" ans is -1";
                return 0;
            }
            maxDist = maxH.top();
            maxH.pop();
            cnt++;
        }
        maxH.push(intervals[i][1]);
        i++;
    }

    if(maxDist < target && (maxH.empty() || maxH.top() < target))
    {
        cout<<" ans is -1";
        return 0;
    }

    if(maxDist < target && maxH.top() >= target)
    {
        cnt++;
        maxDist=maxH.top();
        maxH.pop();
    }

    if(maxDist < target)
    {
        cout<<" ans is -1";
        return 0;
    }

    cout<<"ans is "<<cnt;
    return 0;
}
```

**Optimal Solution (if different):** Not revealed — Aayush's solution is optimal.

**Time Complexity:** O(n log n)
**Space Complexity:** O(n)

---

## Feedback Given

### Problem: Minimum Number of Intervals to Cover Target Range
**Topic:** Greedy + Heap | **Difficulty:** Medium-Hard

### 1. Problem Understanding & Clarification — 2/5
Dove straight into the approach without asking a single clarifying question. Missed opportunities:
- Are endpoints inclusive?
- Can intervals extend beyond target?
- Are duplicate intervals possible?
- Is there a constraint on interval values (overflow risk)?

Recurring pattern — costs real points in interviews where assumptions can sink an otherwise correct solution.

### 2. Approach & Thought Process — 4/5
Strong performance:
- Good instinct to start with recursion and define state clearly
- Immediately identified the validity constraint (left[i] ≤ maxDist) when probed
- Correctly recognized the state-space blowup and pivoted to greedy + heap without much prompting
- Post-loop termination reasoning was sound — correctly deduced only one pop is needed

Minor deduction: needed a prompt to think about the post-loop case.

### 3. Code Quality & Correctness — 3/5
Structure was clean and readable. However:
- **Bug:** Unconditional cnt++ at the end was incorrect — when maxDist >= target at loop exit, no additional interval is needed. Not caught independently; required a prompted trace-through.
- Fix was correct and fast once pointed out.
- Edge cases like "first interval doesn't start at 0" were handled implicitly but never explicitly mentioned.

### 4. Complexity Analysis — 5/5
Correct and complete: O(n log n) time, O(n) space. No issues.

### 5. Communication — 3/5
- Thought-aloud was decent during the approach phase
- Did not volunteer edge cases before coding
- Did not proactively mention key invariants (heap elements always have left ≤ maxDist, first interval must reach 0)

### Overall: 17/25

**Strengths:** Greedy insight solid; complexity analysis clean; code structure good; fix was quick.
**Focus areas:** Always ask 2–3 clarifying questions before coding. Always trace at least one edge case on your own before declaring done.
