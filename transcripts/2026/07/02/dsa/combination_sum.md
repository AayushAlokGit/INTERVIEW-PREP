# DSA Round Transcript
**Date:** 2026-07-02
**Start Time:** 12:14
**End Time:** 12:42
**Duration:** 28 minutes
**Problem:** Combination Sum
**Topic:** Backtracking / Recursion
**Difficulty:** Medium

---

## Problem Statement
Given an array of distinct integers `candidates` and a target integer `target`, return a
list of all unique combinations of `candidates` where the chosen numbers sum to `target`.
The same number may be chosen an unlimited number of times. Two combinations are unique if
the frequency of at least one chosen number differs. Return combinations in any order.

**Example 1:**
```
Input:  candidates = [2, 3, 6, 7], target = 7
Output: [[2, 2, 3], [7]]
```

**Example 2:**
```
Input:  candidates = [2, 3, 5], target = 8
Output: [[2, 2, 2, 2], [2, 3, 3], [3, 5]]
```

**Constraints:**
- 1 <= candidates.length <= 30
- 2 <= candidates[i] <= 40
- All elements of candidates are distinct.
- 1 <= target <= 40
- Number of unique combinations that sum to target is < 150 for the given input.

---

## Conversation Log

**Interviewer:** (Presented the problem, asked for start time and clarifying questions.)

**Aayush:** 12:14 (start time)

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints — [1,30] candidates, values [2,40] distinct, target [1,40], < 150 combinations guaranteed.

**Aayush:** A brute force approach: sort the candidates, then at each index choose whether to include the current element or not. Any element can be included multiple times as long as target >= candidates[i]. Time complexity ~ 2^(max height of recursion tree), where max height = max(target/minCandidate, number of candidates).

**Interviewer:** Confirmed the include/exclude structure (stay at i on include, advance to i+1 on exclude prevents duplicates). Asked for (1) base cases and (2) whether sorting buys pruning.

**Aayush:** Base cases: (1) target < 0 or elements consumed → return; (2) if target == 0 record current combination into a set for uniqueness. Sorting buys pruning opportunities.

**Interviewer:** Challenged: (1) is a set actually needed given the forward-only index traversal? (2) be concrete about what the pruning is.

**Aayush:** (2) When target < candidates[i] we can return early. (1) Yes, all combinations will be unique because elements are distinct and once we move past an element it's never encountered again — so no set needed.

**Interviewer:** Confirmed — with sorted array, `break` once candidates[i] > target; no set needed. Asked for full code.

**Aayush:** (Submitted solution — see Solution section.)

**Interviewer:** Asked him to verify two things: (1) base-case ordering — `if(target<0 || i==nums.size()) return;` runs before the `target==0` check, so if a call has target==0 AND i==size it returns without recording; is that a real bug? (2) dry-run [2,3,5], target=8 for no duplicates.

**Aayush:** (1) If i == nums.size() there are no elements left, so it doesn't matter what target is; in the current code if solve ever gets target==0 and i==size it's fine to add nothing to res. (2) {2,2,2,2}: i=0 always, target→0, return. {2,3,3}: i=0,i=1,i=1, target=0, return. {3,5}: skip i=0, i=1, i=2, target=0, return.

**Interviewer:** Corrected the reasoning: "fine to add nothing" is WRONG — if target==0 the combination is valid and skipping it would drop a real answer. The real reason there's no bug is that the state is UNREACHABLE: target only decreases via the include branch which keeps i unchanged (< size), so the moment target hits 0 the call has i < size, records, and returns immediately — never recursing further. Confirmed code is correct. Asked for time and space complexity.

**Aayush:** Time complexity ~ 2^(max height of recursion tree), max height = max(target/minCandidate, number of candidates).

**Interviewer:** Pushed on precision: on a single root-to-leaf path both excludes (i up to n times) and includes (target drops up to target/min times) can interleave — so is the max path length max(n, target/min) or n + target/min?

**Aayush:** n + target/min.

**Interviewer:** Confirmed — time O(2^(n + target/min)). Asked for space complexity, separating recursion stack from output storage.

**Aayush:** Recursion stack is n + target/min; output storage is separate.

**Interviewer:** Confirmed — auxiliary space O(n + target/min), output counted separately. Noted solution is a clean optimal formulation; sorting + break only trims constants. Moved to wrap-up and requested end time.

**Aayush:** 12:42 (end time)

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

void solve(vector<int> &nums, int i, int target, vector<int> &cur, vector<vector<int>> &res)
{
    if(target < 0 || i == nums.size()) return;

    if(target == 0)
    {
        res.push_back(cur);
        return;
    }
    // exclude ith element
    solve(nums, i+1, target, cur, res);

    // include ith element
    if(target >= nums[i])
    {
        cur.push_back(nums[i]);
        solve(nums, i, target - nums[i], cur, res);
        cur.pop_back();
    }
}

int main() {
    vector<int> nums{2,3,5};
    int target = 8;
    vector<vector<int>> res;
    vector<int> cur;
    solve(nums, 0, target, cur, res);
    for(auto vec : res)
    {
        for(auto i : vec) cout << i << " ";
        cout << endl;
    }
}
```

**Optimal Solution (if different):** Same approach — this is the standard backtracking solution. A minor variant sorts `candidates` and uses a `for` loop from a start index with `break` when `candidates[i] > target` for pruning; same asymptotic complexity.

**Time Complexity:** O(2^(n + target/min)) — recursion tree height is n (excludes) + target/min (includes) along a single path.
**Space Complexity:** O(n + target/min) auxiliary (recursion stack + `cur`); output storage counted separately.

---

## Feedback Given

### What went well
- Clean, correct code on the first attempt — textbook include/exclude, no duplicates, no bugs.
- Reasoned about the `set` instead of adding it defensively — correctly derived uniqueness from forward-only traversal and dropped it (improvement over prior "add a guard" default).
- Corrected the complexity bound when pushed — recognized a single path interleaves includes and excludes, so height = n + target/min, not max(n, target/min). Caught the "ignores combined worst case" trap.
- Precise space breakdown: recursion stack O(n + target/min), output separate.
- Pace: 28 minutes — right in target zone for a medium.

### What to work on
1. Base-case justification was wrong before it was right. First instinct "it's fine to add nothing" is a FALSE justification — if the state were reachable, skipping would drop a valid answer. Correct reasoning is that the state is unreachable. Separate "if this branch fired, would it be correct?" from "does this branch fire?"

### Scoring (out of 5)
| Category | Score | Notes |
|---|---|---|
| Problem Understanding & Clarification | 4.5/5 | Asked constraints; well-scoped. Could've stated base cases unprompted. |
| Approach & Thought Process | 5/5 | Correct backtracking; reasoned about uniqueness and pruning. |
| Code Quality & Correctness | 5/5 | Correct first pass, clean, no bugs. |
| Complexity Analysis | 4.5/5 | Initial height bound imprecise (max vs sum), self-corrected cleanly; space precise. |
| Communication | 4/5 | Much better verification engagement; one wrong-but-corrected justification. |

**Overall: ~4.6/5** — a noticeably stronger round than the previous one. Verification discipline visibly improving. Keep sharpening the quality of the "why" — separate reachability from correctness-if-reached.

**Time Taken: 28 minutes**
