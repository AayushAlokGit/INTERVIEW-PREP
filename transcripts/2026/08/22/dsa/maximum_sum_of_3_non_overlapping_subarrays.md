# DSA Round Transcript
**Date:** 2026-08-22
**Start Time:** 12:02:28 · **End Time:** 12:51:32 · **Duration:** 49 min
**Problem:** Maximum Sum of 3 Non-Overlapping Subarrays
**Topic:** DP / prefix sums — fix the middle index, precompute best-left and best-right
**Difficulty:** Medium-Hard
**Performance Rating:** 1/5  <!-- machine-read on future rounds; ≤2 = eligible for re-ask, ≥3 retired -->
**Hints Used:** 2/2
**Constraints Asked:** one generic "what are the constraints?" at +2 min (got n ≤ 2·10^4, 1 ≤ nums[i] < 2^16 strictly positive, k ≤ floor(n/3)) · **Never Asked:** what "lexicographically smallest" means when two triples tie on sum — the one semantic that decides accepted vs. wrong-answer; whether values could be negative (volunteered, never used)
**Self-Verified:** N/A — no code was written
**Would it have fit a real 45-min round?** No — cut off inside the approach phase around minute 20, with only a brute force and index ranges produced; zero lines of code

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Clarify | 4 min | 2 min | −2 min | ✅ ahead |
| Approach + dry run | 15 min | never settled | — | ❌ |
| Code complete | 35 min | never written | — | ❌ |
| Test + complexity | 42 min | never reached | — | ❌ |
| **Total** | 45 min | 49 min (abandoned) | +4 min | ❌ |

---

## Problem Statement

You are given an integer array `nums` and an integer `k`.

Find **three non-overlapping subarrays of length `k`** with maximum total sum, and return the result as a list of the three starting indices, in increasing order.

If there are multiple answers, return the **lexicographically smallest** list of indices.

**Example 1:**
```
Input:  nums = [1,2,1,2,6,7,5,1], k = 2
Output: [0,3,5]
```
Explanation: The subarrays are `[1,2]` (index 0), `[2,6]` (index 3), and `[7,5]` (index 5). Total sum = 3 + 8 + 12 = 23.
`[3,5,6]` would also give a valid set of three non-overlapping subarrays, but `[0,3,5]` is lexicographically smaller.

**Example 2:**
```
Input:  nums = [1,2,1,2,1,2,1,2,1], k = 2
Output: [0,2,4]
```

Constraints were withheld. Given on request: `1 <= nums.length <= 2 * 10^4`, `1 <= nums[i] < 2^16` (strictly positive), `1 <= k <= floor(nums.length / 3)`.

---

## Conversation Log

**Interviewer [12:02:28 · +0m]:** Presented the problem as Medium-Hard with the Medium-Hard reference timeline (clarify 4 / approach 15 / code 35 / test 42), stated as measured but not enforced. No constraints given up front. Asked for clarifying questions.

**Aayush [+2m]:** what are the constraints ?

**Interviewer [12:04:16 · +2m]:** `1 <= nums.length <= 2 * 10^4`; `1 <= nums[i] < 2^16` — all values strictly positive; `1 <= k <= floor(nums.length / 3)`, so a valid answer always exists. Anything else?

**Aayush [+26m, after 24 minutes of silence]:** we can consider i1, i2 and i3 to be the required 3 indices then i1 can lie in `[0 : n-3k]`, i2 belongs to `[k, n-2k]` and i3 belongs to `[2k, n-k]`

**Interviewer [12:28:43 · +26m]:** Okay. What's your approach?

**Aayush [+28m]:** brute force could be to fix i1, i2 and i3 and use a prefix sum array to find all possible combinations and then return the lexicographical sequence with maximum sum. We would need a set to track all triplets to ensure ordering of triplets. the TC would be O(n^3)

**Interviewer [12:30:51 · +28m]:** You've got `n` up to `2 * 10^4`. Is `O(n^3)` acceptable here?

**Aayush [+32m]:** no need to reduce the redundant work of checking each combination of i1, i2 and i3 using the constraints logic to remove the need for evaluating all triplets

**Interviewer [12:34:54 · +32m]:** Make that concrete for me. What's the actual algorithm?

**Aayush [+41m]:** 2 pointer approach could be used ? i1 = 0 and i3 = n-k then i2 will be the maximum subarray of length k between i1+k and i3-k right ? Then we can move i1 to the right and i3 to the left to check if the sum increases or not. I don't think this is correct as the movement of pointers is unclear.
*(second message, same turn)* not sure how to reduce complexity can you give a hint

**Interviewer [12:43:30 · +41m]:** *(escalation step 1 — question about his own idea, no hint spent)* Stay on your own idea. You fixed `i1 = 0` and `i3 = n-k` and maximized `i2` between them, then said the pointer movement is unclear. Construct me two small arrays with the same `k`: one where sliding `i1` one step right **increases** the best total, and one where it **decreases** it. What does having both tell you about the two-pointer plan?

**Aayush [+47m]:** not able to formulate the algorithm

**Interviewer [12:49:24 · +47m]:** *(HINT 1 — directional)* You don't need to kill all three loops. Killing two is enough — one loop over a single index, with everything else answered in `O(1)` at each step. So: keep exactly one of `i1`, `i2`, `i3` as the loop variable, and make the other two fall out for free. Which one do you keep, and what would you need precomputed to make the other two free?

**Aayush [+48m]:** not getting any ideas

**Interviewer [12:50:16 · +48m]:** *(HINT 2 — the key insight)* Loop over the **middle** index `i2`. Once `i2` is fixed, everything to the left of it and everything to the right of it are two completely independent problems: the best single window of length `k` ending anywhere in `[0, i2-k]`, and the best single window of length `k` starting anywhere in `[i2+k, n-k]`. Both can be precomputed with a single pass each, before the main loop. Take it from there — that's all you're getting.

**Aayush [+49m]:** need more help

**Interviewer [12:51:18 · +49m]:** That's everything I can give you. Put down the best thing you have — code whatever you can from where you are, even if it's partial.

**Aayush [+49m]:** end this round and give debrief

**Interviewer [12:51:32 · +49m]:** Round closed with no code submitted.

---

## Solution

**Aayush's Final Solution:** none submitted.

What he produced:
1. Index ranges (correct): `i1 ∈ [0, n-3k]`, `i2 ∈ [k, n-2k]`, `i3 ∈ [2k, n-k]`.
2. Brute force (correct, correctly priced): fix all three indices, price each window with a prefix-sum array, `O(n^3)`.
3. A two-pointer sketch he rejected himself as unsound ("the movement of pointers is unclear").

**Optimal Solution:**
```cpp
vector<int> maxSumOfThreeSubarrays(vector<int>& nums, int k) {
    int n = nums.size();
    vector<long long> pre(n + 1, 0);
    for (int i = 0; i < n; ++i) pre[i+1] = pre[i] + nums[i];

    int m = n - k + 1;                       // number of valid window starts
    vector<long long> w(m);
    for (int i = 0; i < m; ++i) w[i] = pre[i+k] - pre[i];

    vector<int> left(m), right(m);
    int best = 0;
    for (int i = 0; i < m; ++i) {            // strict >  -> keeps the SMALLEST index on ties
        if (w[i] > w[best]) best = i;
        left[i] = best;
    }
    best = m - 1;
    for (int i = m - 1; i >= 0; --i) {       // >= while scanning backward -> also smallest index
        if (w[i] >= w[best]) best = i;
        right[i] = best;
    }

    vector<int> ans;
    long long bestTotal = -1;
    for (int mid = k; mid <= m - 1 - k; ++mid) {
        int l = left[mid - k], r = right[mid + k];
        long long tot = w[l] + w[mid] + w[r];
        if (tot > bestTotal) { bestTotal = tot; ans = {l, mid, r}; }
    }
    return ans;
}
```
Verified: `[1,2,1,2,6,7,5,1], k=2` → `[0,3,5]`; `[1,2,1,2,1,2,1,2,1], k=2` → `[0,2,4]`.

**Time Complexity:** optimal is O(n) — his answer: O(n^3) for the brute force only, no optimal reached · **Space Complexity:** never discussed

---

## Feedback Given

### Round Conditions

- **Hints used: 2/2.** Ceiling: max 2/5. First was "keep one of the three indices as your loop variable and make the other two free"; second was the actual insight — loop the middle, precompute left and right. Both requested, not volunteered.
- **Constraints asked:** one generic "what are the constraints?" at +2 min. Got `n ≤ 2·10^4`, `1 ≤ nums[i] < 2^16` (strictly positive), `k ≤ ⌊n/3⌋`.
- **Never asked:** what "lexicographically smallest" means when two different triples tie on sum — the difference between an accepted solution and a wrong-answer verdict on this problem. Also never asked whether values could be negative (volunteered, never used).
- **Self-verified: N/A.** No code was written.

### Rubric

**Problem understanding & clarification — 2/5.** One generic catch-all question. Good data back, almost none of it used: `n ≤ 2·10^4` sat in hand for 26 minutes before he had to be asked whether `O(n^3)` fit inside it, and the lexicographic rule — the one genuinely tricky part of the spec — was never probed.

**Approach & thought process — 1/5.** The index-range derivation at +26 min was correct and worth something. The brute force was correctly stated and correctly priced at `O(n^3)`. After that, nothing moved. "Reduce the redundant work using the constraints logic" is a restatement of the goal, not a step toward it. The two-pointer sketch was the wrong family entirely — he knew it, which counts for something, but knowing an idea is wrong isn't the same as generating a different one.

**Code quality & correctness — N/A.** No code.

**Complexity analysis — 2/5.** Correctly identified the brute force as `O(n^3)` unprompted — real credit. But didn't check it against a budget already in hand. Asked "is `O(n^3)` acceptable with `n = 2·10^4`?" he answered instantly and correctly. The arithmetic isn't the problem; running it unprompted is.

**Communication — 1/5.** The 24-minute silence from +2 to +26 is the defining event. Constraints at minute 2, nothing until minute 26, and what emerged was index bookkeeping, not an approach. Then three consecutive turns of "not able to formulate" / "not getting any ideas" / "need more help", none of which conveyed where he actually was. Asked to build two concrete counterexamples for his own two-pointer idea, he didn't attempt it — he asked for more help, which spent the second hint.

**Time management — 1/5.** See pace report.

### Pace Report

*(table as above)*

**Would this have fit a real 45-minute round? No — and not marginally.** A real interviewer's cut-off lands inside the approach phase, around minute 20, with only a brute force and index ranges produced. Zero lines of code. The interviewer's note: *"stated brute force correctly, could not optimize, required the key insight to be given, no implementation."* Clear no-hire on a Medium-Hard.

**Biggest time sink: the 24-minute opening silence** — more than half a real round, before the first substantive sentence, producing index bookkeeping rather than an approach. Second sink: the 9-minute gap from +32 to +41 ending in a hint request.

**Cross-round observation:** this is the second round today failing in the same shape — long silence, then a request for help, rather than visible incremental progress. On Push Dominoes the 23-minute silence still produced correct code, hiding the pace cost. Here there was no code to hide behind and the same habit produced a 1. The habit didn't change between rounds; only the problem's tolerance for it did.

### Performance Rating: 1/5

No working solution, and the core insight had to be handed over in full. Two hints used, which would cap at 2 — but the cap isn't binding. The score is 1 on merit: nothing submitted.

### Algorithmic Thought-Process Debrief

**1. The derivation chain**

*Brute force.* Three nested loops over `i1 < i2 < i3` with a prefix-sum array to price each window in `O(1)`. `O(n^3)`. He had this at +28 min.

*Step 0 — price it against the budget.* `n = 2·10^4` → `n^3 = 8·10^12`. Dead by ~10^7. `n^2 = 4·10^8` also too slow. Target is `O(n)` or `O(n log n)`. This line should have been run at minute 26, the moment `O(n^3)` was written; it tells you the answer is a linear scan, which enormously narrows the hunt.

*Trigger 1 — name the redundant work.* For a fixed `i2`, the inner loops re-scan the same left region and the same right region for every different `(i1, i3)` pair. The best `i1` for a given `i2` does not depend on `i3` at all. That independence is the whole problem.

*Trigger 2 — which variable, once fixed, decouples the rest?* The unlocking question, never run. Test all three:
- Fix `i1`: `i2` and `i3` still interact (`i3 ≥ i2 + k`). Not decoupled.
- Fix `i3`: same, `i1` and `i2` still interact. Not decoupled.
- **Fix `i2`**: `i1` is constrained only by `i1 ≤ i2 - k`, `i3` only by `i3 ≥ i2 + k`. Opposite sides of a wall, cannot see each other. **Fully decoupled.**

The middle element is the only one whose removal splits the remainder into two independent halves. Not a coincidence about this problem — it's what "middle" means structurally, and why "fix the middle" is a standing move whenever three ordered things must be chosen.

*Trigger 3 — can each side be made `O(1)`?* For fixed `i2`, you need "best window start in the prefix `[0, i2-k]`" and "best window start in the suffix `[i2+k, n-k]`". Both are running extrema over a prefix / suffix — the classic one-pass precompute.

*Trigger 4 — the tie-break, where this problem actually bites.* "Lexicographically smallest" dictates three specific comparison operators:
- `left[]`: strict `>` scanning forward — an equal-sum later window never displaces an earlier one. Smallest `i1` wins.
- `right[]`: `>=` scanning **backward** — an equal-sum earlier window *does* displace the later one. Smallest `i3` wins.
- Main loop: iterate `mid` ascending with strict `>` on the total. Smallest `i2` survives a tie.

Get any one backwards and you fail on `[1,2,1,2,1,2,1,2,1]` — the all-ties case, which is exactly why that's example 2.

*Final:* three linear passes, `O(n)` time, `O(n)` space.

**2. The signal he missed**

He was one question away at minute 32, and the question was inside the sentence he'd already written: *"i1 can lie in `[0, n-3k]`, i2 belongs to `[k, n-2k]`, i3 belongs to `[2k, n-k]`."* He had already noticed the three indices are separated by walls of width `k`. The next question — *"if I nail one down, do the other two still constrain each other?"* — is a mechanical follow-up to his own observation with only one interesting answer.

Instead he jumped to two pointers, a technique that assumes the objective is **monotone** in pointer movement. It isn't: sliding `i1` right can raise or lower the total depending on what's under the new window. He sensed it — "the movement of pointers is unclear" — without converting the sense into a rejection and a redirect. Asked to construct the two arrays that would prove non-monotonicity, that was the moment to kill two pointers and go hunting elsewhere. He asked for a hint instead.

That's `Q3` — *fix the most constrained variable and query the rest* — the second time it has been the unlocking question and gone unasked.

**3. The generalization**

*Class:* choose `j` ordered, non-overlapping items from a line, maximizing total.

*The move:* enumerate the **middle** choice; precompute "best on the left of here" by forward scan and "best on the right of here" by backward scan. `O(n)` per additional element. Generalizes to `j` subarrays as `dp[t][i] = max(dp[t][i-1], dp[t-1][i-k] + w[i])` — the same idea with the prefix-max folded into the table.

*Family members already seen:* Best Time to Buy and Sell Stock III/IV (fix the split day, best-profit-left + best-profit-right — he did IV on 07/05); Product of Array Except Self (left-carry × right-carry); Trapping Rain Water (max-left, max-right per index). Same skeleton each time: a per-index answer needing one fact about everything before it and one fact about everything after it.

*The tell:* three or more ordered choices with separation constraints, where the outer two don't interact once the middle is fixed. Enumerate the middle. Don't reach for two pointers — two pointers requires a monotone objective, and "pick the best window somewhere over there" is not monotone.

*Second tell, more general:* when a brute force has `j` nested loops, don't try to eliminate all `j`. Ask which **single** loop to keep and what precomputation makes the rest `O(1)`. That reframing — *kill `j-1` loops, not `j`* — is free to ask on every optimization problem.

**4. One concrete drill**

*Part A (15 min) — implement this problem from scratch, no notes.* Include all three tie-break operators; test on `[1,2,1,2,1,2,1,2,1], k=2`. Not getting `[0,2,4]` means one of the three comparisons has the wrong strictness. Then do LC 123 Best Time to Buy and Sell Stock III the same way — fix the split point, best-left + best-right — and notice the identical skeleton.

*Part B — the one that matters, and it isn't about arrays.* Three consecutive rounds have failed the same way: long silence, then a request for help. Hard rule:

> **Every 3 minutes, say one sentence out loud, whatever state you're in.**

Not a solution — a status. "Brute force is `O(n^3)`, that's dead at `n=2e4`, so I need linear." "I'm testing whether fixing `i1` decouples the other two — it doesn't." "Two pointers needs monotonicity and I don't think I have it; let me check with a small array."

Three things happen. The interviewer sees work happening and keeps you alive. Articulating a dead end is what makes you abandon it — he sat on two pointers for nine minutes partly because he never said out loud why it was wrong. And the sentences generate the next question: "fixing `i1` doesn't decouple them" said aloud almost forces "does fixing `i2`?" — which is the entire solution.

He is not failing these rounds on algorithmic knowledge. He priced `O(n^3)` correctly, derived the index ranges correctly, and correctly diagnosed his own two-pointer idea as unsound. He is failing them by doing all of that silently and then stopping.
