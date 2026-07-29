# DSA Round Transcript
**Date:** 2026-06-30
**Start Time:** 9:57
**End Time:** 10:29
**Duration:** 32 minutes
**Problem:** Minimum Number of Arrows to Burst Balloons
**Topic:** Greedy / Intervals
**Difficulty:** Medium

---

## Problem Statement
There are some spherical balloons taped onto a flat wall that represents the XY-plane. The balloons are represented as a 2D integer array `points` where `points[i] = [xstart, xend]` denotes a balloon whose horizontal diameter stretches between `xstart` and `xend`.

Arrows can be shot up directly vertically (in the positive y-direction) from different points along the x-axis. A balloon with `[xstart, xend]` is burst by an arrow shot at `x` if `xstart <= x <= xend`. There is no limit to the number of arrows that can be shot. A shot arrow keeps traveling up infinitely, bursting any balloons in its path.

Given the array `points`, return the minimum number of arrows that must be shot to burst all balloons.

**Example 1:**
```
Input: points = [[10,16],[2,8],[1,6],[7,12]]
Output: 2
Explanation: One arrow at x=6 (bursts [2,8] and [1,6]); one arrow at x=11 (bursts [10,16] and [7,12]).
```

**Example 2:**
```
Input: points = [[1,2],[3,4],[5,6],[7,8]]
Output: 4
Explanation: No two balloons overlap, so one arrow each.
```

**Constraints:**
- `1 <= points.length <= 10^5`
- `points[i].length == 2`
- `-2^31 <= xstart < xend <= 2^31 - 1`

---

## Conversation Log

**Interviewer:** Presented the problem; asked Aayush to note start time and raise any clarifying questions.

**Aayush:** 9:57 (start time).

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints (n up to 1e5, values in [-2^31, 2^31 - 1], xstart < xend). Noted there are things worth observing in there.

**Aayush:** What about the vertical alignment of the balloons?

**Interviewer:** Vertical position is irrelevant — arrows travel straight up infinitely, so only x-interval overlap matters. It's a pure 1D interval problem.

**Aayush:** When 2 intervals overlap the number of arrows reduces, so to minimise arrows we fuse as many intervals as possible. When 2 intervals fuse, their intersection is produced; any arrow fired in the fused intersection bursts both original balloons. Sort intervals by start time and fuse one by one. Maintain a vector `v` of fused intervals; `v[0] = intervals[0]`. For i=1..n-1, if `interval[i]` overlaps `v.back()`, pop_back, fuse, and insert.

**Interviewer:** Solid greedy. Define "overlapping" precisely — do `[1,2]` and `[2,3]` overlap? What's the exact condition?

**Aayush:** They overlap. Overlap condition is `int1.end <= int2.start`.

**Interviewer:** Test that on `[1,6]` and `[2,8]` (sorted by start). They overlap, but `6 <= 2` is false. The inequality is off — re-examine.

**Aayush:** Sorry, reverse — `int2.start <= int1.end`.

**Interviewer:** Correct, and the `<=` makes the touching case count as overlapping. Write the full solution.

**Aayush:** (C++ solution below.) TC = O(n log n) for sorting + O(n) iteration; SC = O(n) for the intersections vector, reducible to O(1) if we only keep the last interval.

**Interviewer:** Dry-ran his code on `{{1,2},{3,4},{1,6},{7,8}}` → sorted `{1,2},{1,6},{3,4},{7,8}` → answer 3 (correct). Asked: (1) convince me the greedy is optimal (lower bound), and (2) edge cases.

**Aayush:** One arrow can break multiple balloons, so overlapping intervals share an arrow. I have covered all the cases.

**Interviewer:** That's the upper bound only; need the lower bound. Also back up "covered all cases" against single balloon, identical balloons, and the 2^31 value range.

**Aayush:** Single point works; identical points intersect to the same interval so it works; values are within int range so no overflow.

**Interviewer:** Correct on edges (also flagged that a naive midpoint `(start+end)/2` would overflow). But you dropped the optimality lower-bound question — re-asked: when forced to fire a new arrow, the first interval of each group is pairwise non-overlapping; what does K such groups imply?

**Aayush:** The intervals are disjoint so each needs its own arrow.

**Interviewer:** Exactly — K disjoint witnesses ⇒ any solution needs ≥ K arrows (lower bound); algorithm achieves K (upper bound) ⇒ optimal. Asked for end time.

**Aayush:** 10:29 (end time).

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

bool isOverlapping(vector<int> &int1, vector<int> &int2) {
    // int[0] - Start, int[1] - end
    return int2[0] <= int1[1];
}

int main() {
    vector<vector<int>> points{{1,2},{3,4},{1,6},{7,8}};
    int n = points.size();
    vector<vector<int>> v;
    sort(points.begin(), points.end());

    v.push_back(points[0]);

    for (int i = 1; i < n; i++) {
        bool overlapping = isOverlapping(v.back(), points[i]);
        vector<int> intervalToInsert = points[i];
        if (overlapping) {
            intervalToInsert[0] = max(points[i][0], v.back()[0]);
            intervalToInsert[1] = min(points[i][1], v.back()[1]);
            v.pop_back();
        }
        v.push_back(intervalToInsert);
    }
    cout << "Ans is " << v.size();
    return 0;
}
```

**Optimal Solution (same complexity, O(1) space — sort by end variant):**
```cpp
int findMinArrowShots(vector<vector<int>>& points) {
    sort(points.begin(), points.end(),
         [](auto& a, auto& b){ return a[1] < b[1]; });
    int arrows = 1;
    long long curEnd = points[0][1];
    for (int i = 1; i < (int)points.size(); i++) {
        if (points[i][0] > curEnd) {   // no overlap with current arrow
            arrows++;
            curEnd = points[i][1];
        }
    }
    return arrows;
}
```
Aayush's running-intersection (sort by start) approach is equally correct; the sort-by-end variant just tracks the right edge directly for O(1) space.

**Time Complexity:** O(n log n) — correct
**Space Complexity:** O(n), reducible to O(1) — correct, volunteered

---

## Feedback Given

**Overall:** Strong round. Found the right greedy quickly, correct code on first real pass, precise complexity, volunteered the O(1) space optimization unprompted. Gaps were in rigor of communication — backing claims vs. asserting them.

### Scoring

**Problem Understanding & Clarification — 4/5**
Asked good questions (constraints, vertical alignment — probing real structure). Ding: didn't proactively pin down the inclusive/touching-boundary case; interviewer had to surface it.

**Approach & Thought Process — 4.5/5**
Clean reduction to a 1D interval problem and correct running-intersection greedy. Reasoned about why fusing reduces arrows from the start. Not a 5 only because the optimality argument needed prompting.

**Code Quality & Correctness — 4/5**
Final code correct and handled all edge cases. But stated the overlap condition backwards first (`int1.end <= int2.start`) and only caught it when asked to test against `[1,6]`/`[2,8]`. A self dry-run of his own stated condition would have caught it.

**Complexity Analysis — 5/5**
O(n log n) time, O(n) → O(1) space. Correct, complete, proactive. Also correctly reasoned no overflow (avoided additions; aware of the midpoint-overflow trap).

**Communication — 3.5/5**
1. Dropped a sub-question twice — answered only one half of a two-part question each time.
2. "I have covered all the cases" — asserted instead of demonstrating; volunteer the trace.
3. First optimality answer gave only the upper bound; missed the lower bound (disjoint witnesses ⇒ ≥K). Got there with a nudge.

**Time Taken: 32 minutes**

### Top takeaway
You solve well; level up the defense. Two habits: (1) when asked multiple things, answer all of them; (2) when you state a condition or claim "done," dry-run it on one concrete example before moving on. That habit catches the reversed inequality and pre-empts half the follow-ups.
