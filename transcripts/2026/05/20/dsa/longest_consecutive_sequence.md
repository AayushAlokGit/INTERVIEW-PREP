# DSA Round Transcript
**Date:** 2026-05-20
**Start Time:** 10:19
**End Time:** 10:37
**Duration:** 18 minutes
**Problem:** Longest Consecutive Sequence
**Topic:** Hash Set / Arrays
**Difficulty:** Medium

---

## Problem Statement
Given an unsorted array of integers `nums`, return the length of the longest consecutive
elements sequence. A consecutive sequence is a run of integers that differ by 1. The
algorithm must run in `O(n)` time.

Example:
```
Input:  nums = [100, 4, 200, 1, 3, 2]
Output: 4   (longest consecutive sequence is [1,2,3,4])
```
```
Input:  nums = [0, 3, 7, 2, 5, 8, 4, 6, 0, 1]
Output: 9
```

Constraints: `0 <= nums.length <= 10^5` (can be empty), `-10^9 <= nums[i] <= 10^9`
(negatives and duplicates allowed).

---

## Conversation Log

**Interviewer:** Presented Longest Consecutive Sequence. Asked for clarifying questions
and approach, stressing the O(n) requirement.

**Aayush:** What are the constraints?

**Interviewer:** Length 0..1e5 (can be empty), values -1e9..1e9, negatives and duplicates
allowed.

**Aayush:** A simple sort of the array will easily give us the longest consecutive sequence.

**Interviewer:** Sorting is O(n log n), but the problem requires O(n). Think of a data
structure giving O(1) membership checks.

**Aayush:** Put elements into a min heap, keep popping; if last popped = current top - 1,
increment consecutive count and update answer, else reset count.

**Interviewer:** Heap doesn't work either — building/popping is O(n log n), and a heap
doesn't give O(1) membership. The structure is a hash set. With O(1) membership I can walk
x+1, x+2, ... but walking from every element is O(n^2). How do you decide which elements
are worth starting a walk from?

**Aayush:** Start a walk only if this is the start of a sequence, i.e. x-1 is not present
in the set.

**Interviewer:** Correct. Each element gets walked at most once, total O(n). Write the code.

**Aayush:** (submitted a version using `set<int>` — see below)

**Interviewer:** Trace it on [100,4,200,1,3,2]; does empty array return 0? Then give time
complexity — look carefully at your data structure choice.

**Aayush:** (resubmitted using `unordered_set<int>`) Time complexity O(n) for iteration and
O(1) for set lookups; space O(n) for the set.

**Interviewer:** You have a while nested in a for — that looks O(n^2). Justify precisely
why it's O(n).

**Aayush:** The x-1 presence check in the set ensures it is not O(n^2).

**Interviewer:** That's the trigger, not the argument. Across the entire run, how many
times total does the inner while body execute? Tie it to the array elements.

**Aayush:** Each element is visited at most twice.

**Interviewer:** Correct — once by the outer for's num-1 check, at most once by an inner
while walk (each element belongs to exactly one run). Inner while executes <= n times
total, so overall O(n). Wrapped up.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int longestConsecutive(vector<int>& nums)
{
    unordered_set<int> st(nums.begin(), nums.end());
    int longest = 0;
    for (int num : st)
    {
        // start of sequence
        if (st.find(num - 1) == st.end())
        {
            int currentNum = num;
            int length = 1;
            while (st.find(currentNum + 1) != st.end())
            {
                currentNum++;
                length++;
            }
            longest = max(longest, length);
        }
    }
    return longest;
}

int main()
{
    vector<int> nums = {100, 4, 200, 1, 3, 2};
    cout << longestConsecutive(nums);
    return 0;
}
```
(First attempt used `set<int>` instead of `unordered_set<int>` — corrected when prompted,
since `std::set` is O(log n) per op and would have made the algorithm O(n log n).)

**Optimal Solution:** Same as the final solution above.

**Time Complexity:** `O(n)` — Aayush's answer, correct. Inner while body executes <= n
times total because each element belongs to exactly one consecutive run.
**Space Complexity:** `O(n)` for the hash set — correct.

---

## Feedback Given

**Time Taken: 18 minutes**

### Scoring

| Criterion | Score | Notes |
|---|---|---|
| Problem Understanding & Clarification | 7.5/10 | Asked for constraints up front. |
| Approach & Thought Process | 4.5/10 | O(n) was stated twice; still opened with sorting (O(n log n)). When pushed, jumped to a min-heap (also O(n log n)) and ignored the explicit O(1)-membership hint. Reached the hash-set approach only after the interviewer named it. |
| Code Quality & Correctness | 6.5/10 | Final code correct — handles empty, duplicates, negatives. But first wrote `std::set` (balanced BST, O(log n) ops), silently breaking the O(n) bound. Fixed to `unordered_set` when prompted. |
| Complexity Analysis | 6.5/10 | Final answer O(n)/O(n) correct, but needed two prompts to articulate the amortized argument for the nested loop. |
| Communication | 6/10 | Terse; reasoning surfaced only under direct questioning, never volunteered. |

**Overall: ~6/10**

### The one thing to fix
Strong default toward generic patterns even when the problem signals the intended approach.
The O(n) requirement IS the hint — it rules out sorting and heaps. When a stated complexity
bound is tighter than the obvious solution, the first move should be "what does this bound
forbid, and what O(1)/O(n) structure does that point to?" — not "let me sort." This repeats
the prior-session weakness of defaulting to generic over structure-exploiting approaches.
