# DSA Round Transcript
**Date:** 2026-06-07
**Start Time:** 13:10
**End Time:** 13:49
**Duration:** 39 minutes
**Problem:** Next Permutation
**Topic:** Arrays / In-place manipulation
**Difficulty:** Medium

---

## Problem Statement
A permutation of an array of integers is any arrangement of its members. The next permutation of an array is the next lexicographically greater arrangement of its elements. If no such greater arrangement exists (array is in descending order), the next permutation is the lowest possible order (sorted ascending). Given an array `nums`, modify it in place to its next permutation, using only constant extra memory.

**Examples:**
- `[1,2,3] -> [1,3,2]`
- `[3,2,1] -> [1,2,3]` (wrap around)
- `[1,1,5] -> [1,5,1]` (duplicates)
- `[1,5,8,4,7,6,5,3,1] -> [1,5,8,5,1,3,4,6,7]` (only tail changes)
- `[5] -> [5]`

**Constraints:**
- `1 <= nums.length <= 100`
- `0 <= nums[i] <= 100`

---

## Conversation Log

**Interviewer:** Presented the problem with examples and asked for clarifying questions.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints (length 1–100, values 0–100); noted duplicates, zeros, and length-1 cases.

**Aayush:** Asked for more examples.

**Interviewer:** Provided several more, including the long case `[1,5,8,4,7,6,5,3,1] -> [1,5,8,5,1,3,4,6,7]` to reveal the structure, plus `[1,3,2]->[2,1,3]`, `[2,3,1]->[3,1,2]`, `[1,5,1]->[5,1,1]`, `[5]->[5]`.

**Aayush:** Approach — find first index `i` from the right where `nums[i] > nums[i-1]`. For a descending array no such `i` exists and `i` becomes 0 → reverse the array. Same for all-equal. When `i > 0`, `pivot = i-1`; swap `nums[pivot]` with the smallest element in `[pivot+1, n-1]` that is greater than `nums[pivot]`; then sort `[pivot+1, n-1]`.

**Interviewer:** Confirmed the approach is correct, played it back, asked him to code it.

**Aayush:** (submitted C++ code — see Solution)  TC O(n log n), SC O(1).

**Interviewer:** Confirmed correct and complexity accurate for that implementation. Pointed at his own comment that the suffix is descending, and asked: (1) do you need a full sort, what's cheaper and what does it do to complexity? (2) prove the suffix stays descending after the swap.

**Aayush:** We can reverse instead of sorting.

**Interviewer:** Confirmed reverse → O(n). Pushed for the justification (question 2): why is the suffix still descending after the swap?

**Aayush:** Because if we swap pivot with j, `nums[j] > nums[pivot]`, and `nums[j-1] > nums[j] > nums[pivot]`, so still descending.

**Interviewer:** Noted that only covers the left neighbor. Asked why `old_pivot >= nums[j+1]` (the right neighbor) — what property of how `j` was chosen guarantees it?

**Aayush:** `j is in [pivot+1, n-1]` (incomplete).

**Interviewer:** Nudged — you scanned `j` from the right and stopped at the first element greater than pivot; what does that say about elements to the right of `j`?

**Aayush:** They are less than or equal to pivot.

**Interviewer:** Confirmed — that closes the proof: right-of-`j` elements are `<= old_pivot`, so the swapped-in value is `>= nums[j+1]`; combined with the left side the suffix stays descending, so reverse is valid. Optimal solution is O(n)/O(1). Wrapped up and gave feedback.

---

## Solution
**Aayush's Final Solution (with the reverse optimization applied):**
```cpp
void nextPermutation(vector<int>& nums) {
    int n = nums.size();
    int i = n - 1;
    while (i > 0 && nums[i] <= nums[i-1]) i--;

    if (i == 0) {                       // descending / all-equal
        reverse(nums.begin(), nums.end());
        return;
    }

    int pivot = i - 1;
    // [pivot+1, n-1] is in descending order
    for (int j = n - 1; j > pivot; j--) {
        if (nums[j] > nums[pivot]) {
            swap(nums[pivot], nums[j]);
            break;
        }
    }
    reverse(nums.begin() + pivot + 1, nums.end());   // suffix still descending -> reverse = ascending
}
```
(As submitted he used `sort(nums.begin()+pivot+1, nums.end())` giving O(n log n); switched to `reverse` for O(n) after prompting.)

**Optimal Solution (if different):** Same as above — optimal.

**Time Complexity:** O(n log n) as first written; O(n) after the reverse optimization.
**Space Complexity:** O(1).

---

## Feedback Given

### What went well
- **Clarification was strong** — asked for constraints *and* more examples, and used the long example to surface the structure (clear improvement).
- **Went straight to the structure-exploiting algorithm** — pivot from the right, swap with smallest larger suffix element, fix suffix. No brute-force permutation generation.
- **Edge cases handled in code** — descending/all-equal (`i==0 -> reverse`) and single element correct from the first write.
- **Complexity accurate** both before and after optimization.

### What to sharpen
- **Noticed the structure but didn't exploit it unprompted** — his own comment said the suffix is descending, yet he called `sort()`. The O(n log n) -> O(n) win (reverse) needed me to point at his comment. When you observe a structural property, immediately ask what it lets you do cheaper.
- **Correctness proof came in halves** — justified the left neighbor confidently but needed two nudges for the right-neighbor argument (right-of-`j` elements are `<= pivot` because of the right-to-left scan). Finish the proof before moving on.
- **Didn't volunteer a dry-run** — asserted complexity and stopped; interviewer traced the edges. Volunteer a quick trace of a tricky case before declaring done.

### Scoring (out of 5)
| Criterion | Score | Notes |
|---|---|---|
| Problem understanding & clarification | 4.5 | Asked constraints *and* examples; used them to find structure |
| Approach & thought process | 4.5 | Went straight to the optimal structural algorithm |
| Code quality & correctness | 4.5 | Correct, clean, edges handled; left-in debug prints, sort over reverse initially |
| Complexity analysis | 5 | Accurate before and after |
| Communication | 3.5 | Optimization and full proof both needed prompting |

**Overall: 4.4 / 5** — Strong, efficient round. Had the algorithm and the structural insight; the gap is acting on structure already spotted and finishing proofs without nudging.

**Time Taken: 39 minutes**
