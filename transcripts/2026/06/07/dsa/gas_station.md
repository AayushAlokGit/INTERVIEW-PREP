# DSA Round Transcript
**Date:** 2026-06-07
**Start Time:** 8:02
**End Time:** 9:40
**Duration:** 98 minutes
**Problem:** Gas Station
**Topic:** Greedy / Prefix Sum
**Difficulty:** Medium

---

## Problem Statement
There are `n` gas stations arranged in a circle. Given two integer arrays:
- `gas[i]` — amount of gas available at station `i`
- `cost[i]` — gas needed to travel from station `i` to the next station `(i+1)`, wrapping around from the last station back to station `0`.

You start with an empty tank at one of the stations and drive clockwise around the circle. Return the starting gas station's index if you can travel around the circuit once in the clockwise direction, otherwise return `-1`. If a solution exists, it is guaranteed to be unique.

**Example:**
```
gas  = [1, 2, 3, 4, 5]
cost = [3, 4, 5, 1, 2]
Output: 3
```
Starting at station 3, the tank never goes negative around the full loop and returns to station 3.

**Constraints:**
- `n == gas.length == cost.length`
- `1 <= n <= 10^5`
- `0 <= gas[i], cost[i] <= 10^4`

---

## Conversation Log

**Interviewer:** Presented the Gas Station problem with example and asked for clarifying questions before approaching.

**Aayush:** What are the constraints?

**Interviewer:** Provided constraints (n up to 1e5, values 0..1e4), noted the solution is unique if it exists and all values are non-negative.

**Aayush:** There can be possible candidates for the starting point — indices where cost of moving to next station <= gas available at current station. For each candidate, simulate moving forward around the circle, deducting cost and adding gas. If at any point you can't move forward, break and mark the candidate invalid. This is brute force, O(n*n) time.

**Interviewer:** Confirmed the wording (tank goes negative => invalid) and that candidate filtering doesn't change asymptotic complexity. Asked if he can do better than O(n²), with a hint about what failing at station `j` tells you about stations between `s` and `j`.

**Aayush:** Asked for a hint.

**Interviewer:** Gave the hint: if you start at `s` and fail at `j`, none of the stations `s..j` can be valid starts, so jump the next candidate to `j+1`. Asked about resulting complexity and how to know up front if any answer exists.

**Aayush:** Reasoned that for a valid start `i`, you complete the trip from `i` to `n-1` then `0` to `i`. Asked whether `sum(gas_j - cost_j)` for `j in [i, n-1]` is the gas remaining when reaching index 0.

**Interviewer:** Confirmed, introduced `diff[j] = gas[j] - cost[j]`, and stated the global fact: a valid start exists iff `sum(diff) >= 0`. Combined with the jump-to-`j+1` insight to describe the O(n)/O(1) single-pass running-tank reset algorithm.

**Aayush:** Asked whether prefix sums on the diff array could be used to find the starting index.

**Interviewer:** Confirmed — the valid start is the index immediately after the global minimum of the prefix-sum curve. Noted this is equivalent to the reset approach. Asked him to code it.

**Aayush:** Asked what the diff array conceptually represents.

**Interviewer:** Explained `diff[i]` is the net gas change for the leg starting at station `i`; the tank is the running sum of diff; total sum = net over the circle; best start is just after the lowest dip.

**Aayush:** Asked if, when constrained to end with exactly 0 fuel it becomes a zero-sum game, whereas allowing surplus forces choosing the global minimum of the prefix diff array.

**Interviewer:** Untangled the conflation — feasibility (any start exists) is governed by `total`, while which start is governed by the prefix minimum (never-go-negative en route). Even in the exact-zero case the post-minimum start is still chosen.

**Aayush:** Asked the diff-array conceptual question again / what it represents (clarified intuition).

**Aayush:** Submitted the solution (see below).

**Interviewer:** Confirmed correctness and clean overflow handling. Asked for two traces (gas=[3,3], cost=[2,2]; and the meaning of the minPrefix=0 / minIndex=-1 initialization) plus complexity.

**Aayush:** Traced correctly that it returns 0, with a small arithmetic slip writing prefix=1 instead of 2 at i=1. Explained minPrefix=0 = no prefix included yet, minIndex=-1 allows the 0th index to be the start.

**Interviewer:** Noted the trace slip (cumulative sum carries over), confirmed the conceptual explanation, asked for complexity.

**Aayush:** O(n) time, O(1) space.

**Interviewer:** Confirmed optimal. Delivered feedback and scoring.

---

## Solution
**Aayush's Final Solution:**
```cpp
class Solution {
public:
    int canCompleteCircuit(vector<int>& gas, vector<int>& cost) {
        int n = gas.size();

        long long prefix = 0;
        long long minPrefix = 0;
        int minIndex = -1;

        for (int i = 0; i < n; i++) {
            prefix += (gas[i] - cost[i]);

            if (prefix < minPrefix) {
                minPrefix = prefix;
                minIndex = i;
            }
        }

        // Total gas < total cost => impossible
        if (prefix < 0)
            return -1;

        return (minIndex + 1) % n;
    }
};
```
**Optimal Solution (if different):** Same as above — optimal.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

---

## Feedback Given

### What went well
- **Clarifying questions** — Asked for constraints up front (a historically skipped area). Next level: proactively raise value-range/overflow implications as clarifications.
- **Honest about brute force** — Stated the O(n²) baseline clearly; identified candidate pruning without overselling it.
- **Drove toward the structural insight** — Connected "global minimum of the prefix curve" to the answer largely on his own; that's the hard part.
- **Clean, correct, optimal code** on first write — O(n)/O(1), overflow-safe, edge cases (n=1, all-positive, impossible) handled.

### What to sharpen
- **Trace arithmetic slipped** — Wrote prefix=1 at i=1 when it should accumulate to 2. Answer unaffected, but a dropped carry in a running sum is a silent bug on harder problems. Re-add each step instead of pattern-matching.
- **Conceptual framing got muddled once** — Conflated the feasibility condition (total >= 0) with the which-start condition (prefix minimum). Recovered, but be precise about which constraint drives which decision.
- **Edge-case volunteering** — Traced when asked, but didn't proactively volunteer n=1 or total==0 cases.

### Scoring (out of 5)
| Criterion | Score | Notes |
|---|---|---|
| Problem understanding & clarification | 4 | Asked for constraints; could probe semantics deeper |
| Approach & thought process | 4.5 | Reached the structural insight largely on his own |
| Code quality & correctness | 5 | Clean, correct, optimal, overflow-safe |
| Complexity analysis | 5 | Immediate and correct |
| Communication | 3.5 | Good dialogue; one trace slip + one muddled framing |

**Overall: 4.4 / 5** — Strong round. Excellent algorithmic instinct; gaps are in precision (trace carefully, frame conditions cleanly).

**Time Taken: 98 minutes**
