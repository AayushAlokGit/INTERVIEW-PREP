# DSA Round Transcript
**Date:** 2026-05-11
**Start Time:** 9:49
**End Time:** 10:44
**Duration:** 55 minutes
**Problem:** Course Schedule III
**Topic:** Greedy, Heap
**Difficulty:** Hard

---

## Problem Statement

There are `n` courses numbered `1` to `n`. You're given an array `courses` where `courses[i] = [duration_i, lastDay_i]` meaning:
- The `i`-th course takes `duration_i` days to complete
- It **must be finished on or before** `lastDay_i`

You **start on day 1** and can only take one course at a time. Each course must be taken **continuously** (no breaks).

Return the **maximum number of courses** you can take.

**Example 1:**
```
courses = [[100, 200], [200, 1300], [1000, 1250], [2000, 3200]]
Output: 3
```

**Example 2:**
```
courses = [[1, 2]]
Output: 1
```

**Example 3:**
```
courses = [[3, 2], [4, 3]]
Output: 0
```

Constraints:
- `1 <= n <= 10^4`
- `1 <= duration_i, lastDay_i <= 10^4`

---

## Conversation Log

**Interviewer:** Please note the current time and share it with me. Here's your problem: [presented Course Schedule III as above]. Do you have any clarifying questions?

**Aayush:** Start time is 9:49. Is the lastDay always >= duration for the ith job?

**Interviewer:** Not necessarily — look at Example 3: [3, 2] has duration 3 but lastDay 2. How will your approach handle those?

**Aayush:** Ok so my approach would be to first drop courses where duration >= lastDay since they can never be taken. Then sort the remaining courses by lastDay. Now at each course we can choose it or not. dp[i] = max number of courses with i courses. dp[i] = dp[i-1] (not taking ith course). dp[i] = dp[j] + 1 where j < i and durationSum[j] <= lastDay[i] - duration[i].

**Interviewer:** Your sort by lastDay is the right foundation. What exactly is durationSum[j]? If dp[j] is the max number of courses from the first j, you may not have taken all of them — so is durationSum the sum of all j durations or only the selected ones?

**Aayush:** durationSum[j] should be the sum of selected courses only. At each time we would also have to keep updating duration[i] depending on which produces max number of courses taking ith course or leaving it.

**Interviewer:** That's the core issue — dp[i] stores only a count, but to make the transition you also need the minimum total time used to achieve that count. Two different subsets of j courses might both give dp[j] = 3 but have different total durations. This suggests rethinking the state — maybe storing minimum time instead of count, or a 2D state dp[i][k].

**Aayush:** A recursion + memoisation: solve(curDuration, i) = max(solve(curDuration + duration[i], i+1) + 1, solve(curDuration, i+1)). Memoize on 2 states curDuration and i.

**Interviewer:** The recurrence is logically correct. But what's the worst-case value of curDuration and how large would the memoization table be?

**Aayush:** 10^8 for curDuration and i = 10^4, total 10^12 — too large to memoize.

**Interviewer:** Exactly. Memoization on raw curDuration is infeasible. What if for a fixed count k you tracked the minimum possible total duration to achieve it? What data structure comes to mind?

**Aayush:** We can use a maxHeap which will have the k courses with lower duration. The sum of these durations would be the answer.

**Interviewer:** You're on the right track. Walk me through the full decision logic — what order do you process courses, when do you add to the heap, and what happens when a course can't fit its deadline?

**Aayush:** Sort by lastDay and ignore courses that can't fit the deadline, push course to heap. At the end keep popping from heap until heap contains k smallest duration courses.

**Interviewer:** There's a gap in the "defer to end" part. Consider [[1,2],[3,4],[2,6]] — if you push all and pop to k smallest, you need to verify deadlines retroactively which doesn't work. When a course can't fit, is there something smarter than skipping it?

**Aayush:** If the selected course in the heap which has the largest duration (maxH.top()) is > duration[i] then we can pop the heap and push duration[i] to minimise total duration of selected jobs, in hopes that in the future we'd be able to schedule more courses.

**Interviewer:** Exactly right — that's the core greedy insight. Swapping keeps count the same but reduces total time. Now code it up.

**Aayush:** And we would iterate k from 0 to n to include all possible max number of courses.

**Interviewer:** Why would you iterate k? The greedy algorithm doesn't need to try different values of k. Just process all courses once, maintain the heap, and what's your answer at the end?

**Aayush:** [coded the solution below]

**Interviewer:** Let me trace Example 1 — correct. What edge cases would you trace before submitting?

**Aayush:** All courses having same lastDay, all courses having lastDay < duration.

**Interviewer:** Those are valid but thin. Your condition is `durationSum + courses[i][0] <= courses[i][1]` — what if they're exactly equal? And what if n=1 with a valid course? And empty input?

**Aayush:** Yes except the input being empty all cases are handled.

**Interviewer:** Trace it — when n=0, sort is a no-op, loop doesn't execute, heap.size() returns 0. Your code does handle empty input. The real gap is you asserted without tracing.

**Aayush:** End time is 10:44.

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<vector<int>> courses{{3, 2}, {4, 3}};
    int n = courses.size();
    priority_queue<vector<int>> maxH;

    // sort by lastDay
    sort(courses.begin(), courses.end(), [](vector<int> &c1, vector<int> &c2){
        return c1[1] < c2[1];
    });

    int durationSum = 0;

    for(int i = 0; i < n; i++) {
        if(durationSum + courses[i][0] <= courses[i][1]) {
            maxH.push(courses[i]);
            durationSum += courses[i][0];
        } else {
            if(!maxH.empty() && courses[i][0] < maxH.top()[0]) {
                auto vec = maxH.top();
                maxH.pop();
                durationSum -= vec[0];
                maxH.push(courses[i]);
                durationSum += courses[i][0];
            }
        }
    }
    cout << "ans is " << maxH.size();
    return 0;
}
```

**Optimal Solution (if different):** Aayush's solution matches the optimal greedy + max heap approach. A minor refinement would be to push only the duration (int) into the heap rather than the full vector, saving minor overhead.

**Time Complexity:** O(n log n)
**Space Complexity:** O(n)

---

## Feedback Given

### 1. Problem Understanding & Clarification — 3/5
Good instinct to question the lastDay >= duration assumption — that led to a real discovery (Example 3 disproves it). However, stopped after one question. Did not ask about: same deadlines, return format edge cases, or value constraints. One good clarifying question, but left it thin.

### 2. Approach & Thought Process — 3/5
Started with a reasonable DP formulation but the state was underspecified — caught the flaw once pressed, showing good reasoning. Independently identified that memoization was infeasible (10^12 states) — strong. However, the greedy + heap swap insight required hints; got to "max heap for k smallest" but needed prompting to see the swap. Once pointed there, understood it cleanly.

### 3. Code Quality & Correctness — 4/5
Clean, idiomatic C++. Correct sort comparator, correct heap usage, correct swap logic, correct answer derivation. No bugs. Minor: pushing the full vector into the heap when only duration[0] is used is slightly wasteful — a heap of just durations would be cleaner.

### 4. Complexity Analysis — 4/5
Correct: O(n log n) time, O(n) space. Slightly imprecise on the breakdown — stating both sorting and heap operations as O(n log n) separately would be more precise — but the answer was right.

### 5. Communication — 2.5/5
Weakest area. When asked for edge cases, gave two surface-level ones and then incorrectly claimed empty input wasn't handled without tracing it first. Recurring pattern — asserting without verifying. Need to volunteer more cases and always trace before claiming.

### Overall: 16.5 / 25

**Strengths:** Recognized DP state flaw independently, caught memoization infeasibility quickly, clean correct code once approach was clear.

**To improve:** Trace edge cases before submitting or asserting. The greedy "swap to defer" pattern with a heap is a recurring gap — practice recognizing it earlier without hints.

**Time Taken: 55 minutes**
