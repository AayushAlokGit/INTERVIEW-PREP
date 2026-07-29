# DSA Round Transcript
**Date:** 2026-05-17
**Start Time:** 8:51
**End Time:** 9:45
**Duration:** 54 minutes
**Problem:** Furthest Building You Can Reach
**Topic:** Heap / Greedy (defer-decision)
**Difficulty:** Medium

---

## Problem Statement
You are given an integer array `heights` representing the heights of buildings, some `bricks`, and some `ladders`.

You start your journey from building `0` and move to the next building by possibly using bricks or ladders.

While moving from building `i` to building `i+1`:
- If the current building's height is >= the next building's height, no resource is needed.
- Else, you can use one ladder OR `(h[i+1] - h[i])` bricks.

Return the furthest building index reachable with optimal use of resources.

**Example 1:** `heights = [4,2,7,6,9,14,12], bricks = 5, ladders = 1` → `4`
**Example 2:** `heights = [4,12,2,7,3,18,20,3,19], bricks = 10, ladders = 2` → `7`

**Constraints:** `1 <= n <= 10^5`, `1 <= heights[i] <= 10^6`, `0 <= bricks <= 10^9`, `0 <= ladders <= n`

---

## Conversation Log

**Interviewer:** Presented problem. Asked for start time and any clarifying questions.

**Aayush:** "8:51". Proposed a local greedy: at each climb, compute % change for bricks vs ladders, pick the lower % (tie-break: lower number).

**Interviewer:** Counterexample — `[1,5,1,100], bricks=4, ladders=1`. Local % rule doesn't know about the future 99-climb.

**Aayush:** Revised — "always try bricks first, then ladders if needed."

**Interviewer:** Counterexample — `[1,2,100,101], bricks=1, ladders=1`. Brick-first burns the brick on a 1-climb, leaving you stuck on a 1-climb after the 98-climb. Hint: think about retroactively swapping a past brick-decision for a ladder.

**Aayush:** "We should defer ladder usage until really needed?"

**Interviewer:** Yes — and when we run out of bricks, swap the **largest** past climb to a ladder and reclaim those bricks. What DS gives largest-so-far efficiently?

**Aayush:** "Max heap of climbs."

**Aayush:** Asked — shouldn't we consider the current climb too?

**Interviewer:** Confirmed — push current climb into heap first, then check.

**Aayush:** Wrote first C++ version using max-heap. Had two bugs:
1. Stop condition `bricks <= 0 && ladders <= 0` placed before spending bricks → could continue with negative bricks.
2. `bricks <= 0` in swap-loop → eagerly swaps to ladder even when bricks=0 (sufficient).

**Interviewer:** Asked him to trace on `[1,5,1,10], bricks=4, ladders=1`.

**Aayush:** "Output 3, correct 3." (Output happened to match optimal but trace ended with negative bricks — invalid state.)

**Interviewer:** Gave a counterexample where the bug actually shows: `[1,5,1,3,1,10], bricks=4, ladders=1` (expected 4; buggy code returns 5).

**Aayush:** Fixed stop-condition (`bricks < climb && ladders==0` before spending). But still had `bricks <= 0` in swap.

**Interviewer:** Pointed out subtle bug: `[1,5,1,100], bricks=4, ladders=1` — eager swap at i=0 wastes ladder.

**Aayush:** Rewrote cleanly — push climb, if `bricks < 0 && ladders > 0` swap, if still `bricks < 0` break.

**Interviewer:** Traced — correct on all inputs. Asked complexity.

**Aayush:** "O(n log n) time, O(n) space."

**Interviewer:** Correct. Asked about space optimization.

**Aayush:** "We only care about the largest climbs."

**Interviewer:** Specifically the `ladders` largest. Use a min-heap of size `ladders`; smaller climbs get popped and paid with bricks. Space: O(ladders).

---

## Solution
**Aayush's Final Solution:**
```cpp
int n = heights.size();
priority_queue<int> maxH;
int i = 0;
for (; i < n - 1; i++) {
    int climb = heights[i + 1] - heights[i];
    if (climb > 0) {
        bricks -= climb;
        maxH.push(climb);
        if (bricks < 0 && ladders > 0) {
            bricks += maxH.top();
            maxH.pop();
            ladders--;
        }
        if (bricks < 0) break;
    }
}
cout << i;
```

**Optimal Solution (min-heap, O(ladders) space):**
```cpp
int furthestBuilding(vector<int>& h, int bricks, int ladders) {
    priority_queue<int, vector<int>, greater<int>> minH;
    int n = h.size();
    for (int i = 0; i < n - 1; i++) {
        int climb = h[i+1] - h[i];
        if (climb <= 0) continue;
        minH.push(climb);
        if ((int)minH.size() > ladders) {
            bricks -= minH.top();
            minH.pop();
        }
        if (bricks < 0) return i;
    }
    return n - 1;
}
```

**Time Complexity:** O(n log n) — or O(n log L) with size-bounded heap
**Space Complexity:** O(n) in his solution, O(L) optimal (L = ladders)

---

## Feedback Given

### Scoring

| Category | Score (/5) | Notes |
|---|---|---|
| Problem Understanding & Clarification | 2 | No clarifying questions — value ranges, behavior on equal heights, etc. Recurring pattern. |
| Approach & Thought Process | 3 | Started with flawed local greedy. Took two counterexamples to reach defer-ladder + heap. Not derived independently. |
| Code Quality & Correctness | 2.5 | First version had `bricks <= 0` bug causing eager ladder use. Required targeted counterexample. Second rewrite clean. |
| Complexity Analysis | 4 | Correct O(n log n) / O(n). |
| Communication | 3.5 | Clear thinking-aloud. Didn't trace edge cases proactively. |

### Key takeaways
1. **Trace before declaring done.** Claimed `[1,5,1,10]` was correct, but trace ended with `bricks=-5` (infeasible) — got right answer by coincidence.
2. **Always ask clarifying questions** — even one frames the problem.
3. **Pattern to internalize:** "Process greedily, use a heap to retroactively swap a past decision when constraints break." (IPO, Maximum Performance of Team, Minimum Refueling Stops.)

**Time Taken: 54 minutes**
