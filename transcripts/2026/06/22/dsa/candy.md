# DSA Round Transcript
**Date:** 2026-06-22
**Start Time:** 16:08
**End Time:** 16:43
**Duration:** 35 minutes
**Problem:** Candy
**Topic:** Greedy / Two-Pass Array
**Difficulty:** Hard

---

## Problem Statement
There are `n` children in a line, each with a rating in `ratings`. Give candies subject to: (1) each child gets at least one candy; (2) a child with a strictly higher rating than a neighbor must get more candies than that neighbor. Return the minimum total candies.

Constraints: `1 <= n <= 2*10^4`, `0 <= ratings[i] <= 2*10^5`. Equal-rated neighbors impose no constraint.

**Example 1:** `ratings = [1,0,2]` → `5` (distribute 2,1,2).
**Example 2:** `ratings = [1,2,2]` → `4` (distribute 1,2,1).

---

## Conversation Log

**Interviewer:** Presented the problem and examples; asked for clarifying questions and approach.

**Aayush:** Start time 16:08. Asked for constraints.

**Interviewer:** Gave constraints (n up to 2e4, ratings 0..2e5, equal allowed, single element possible).

**Aayush:** Asked for a clearer explanation of the higher-rating rule with an example.

**Interviewer:** Explained with `[1,3,2,2,1]` — constraints are local and apply only on strictly-higher ratings; equal ratings impose nothing; both left and right neighbors matter. Showed a minimal distribution 1,2,1,2,1 = 7.

**Aayush:** Proposed two greedy passes. L2R: coins[0]=1; if r[i]>r[i-1], coins[i]=coins[i-1]+1, else 1. R2L: take the max of the existing value and right-neighbor+1, because taking the max respects both left and right effects.

**Interviewer:** Confirmed the approach and the key max insight; restated the precise R2L loop. Asked him to code it.

**Aayush:** Wrote the full C++ solution (coins array init to 1, L2R increment, R2L max, summing during the R2L pass). Tested on `[1,1,1]`. Stated TC O(n), SC O(n), and that SC "can't be reduced further because we need intermediate counts between the two iterations."

**Interviewer:** Verified correctness on `[1,0,2]` → 5 and `[1,1,1]` → 3. Noted `[1,1,1]` is an all-equal masking test that doesn't exercise the R2L max line. Confirmed TC O(n). Challenged the space claim: do you need all n values at once, and can processing as up/down slopes give O(1)? Asked him to compute `[1,2,3,2,1]` via running counters.

**Aayush:** "Not sure how to reduce to O(1)."

**Interviewer:** Gave escalating hint — slopes + arithmetic series; increasing run of length L costs L(L+1)/2; decreasing run mirrors; the shared peak takes the max of both sides. Asked him to try `[1,2,3,2,1]`.

**Aayush:** "Not getting it."

**Interviewer:** Revealed the O(1) one-pass algorithm (up/down/peak counters with the `(peak >= down ? 1 : 0)` adjustment), traced it on `[1,2,3,2,1]` → 9, and explained the peak double-count subtlety. Noted the two-pass O(n) is the standard accepted answer. Moved to feedback.

**Aayush:** End time 16:43.

---

## Solution
**Aayush's Final Solution (O(n) space, correct):**
```cpp
int n = nums.size();
vector<int> coins(n,1);
coins[0] = 1;
for(int i=1;i<n;i++)
    if(nums[i] > nums[i-1]) coins[i] = coins[i-1]+1;
int ans = coins[n-1];
for(int i=n-2;i>=0;i--){
    if(nums[i] > nums[i+1]) coins[i] = max(coins[i], coins[i+1]+1);
    ans += coins[i];
}
return ans;
```

**Optimal (O(1) space, revealed):**
```cpp
int candy(vector<int>& r) {
    int n = r.size(), total = 1, up = 0, down = 0, peak = 0;
    for (int i = 1; i < n; i++) {
        if (r[i] > r[i-1])      { up++; down = 0; peak = up; total += 1 + up; }
        else if (r[i] == r[i-1]){ up = down = peak = 0;      total += 1; }
        else                    { up = 0; down++;            total += 1 + down - (peak >= down ? 1 : 0); }
    }
    return total;
}
```

**Time Complexity:** O(n) (both versions)
**Space Complexity:** O(n) for his two-pass; O(1) for the slope-counting version.

---

## Feedback Given

**Strengths:**
- Reached the two-pass greedy directly, including the crucial R2L `max` insight to respect both neighbors — a structure-exploiting solution arrived at without prompting (historically his weak spot).
- Good clarification engagement: asked for constraints and for a rule example before committing.
- Clean, correct code; verified on examples.

**Areas to improve:**
1. Stopped at "not sure" / "not getting it" on the O(1) optimization without really attempting the scaffolded slope reasoning — the move is to try the small case out loud; even a wrong attempt scores higher than stopping.
2. Asserted SC "can't be reduced further," which was wrong — be careful declaring a bound optimal; honest framing ("I suspect O(1) via slopes but would need to work it out") is stronger.
3. Self-test `[1,1,1]` was a masking input that never exercises the R2L max line — dry-run with the input that stresses the tricky branch (`[1,2,3,2,1]`).

**Scorecard**
| Category | Score |
|---|---|
| Problem Understanding & Clarification | 4.5 / 5 |
| Approach & Thought Process | 4.5 / 5 |
| Code Quality & Correctness | 4.5 / 5 |
| Complexity Analysis | 3.5 / 5 |
| Communication | 3.5 / 5 |

**Overall: 4.1 / 5** — Best DSA round of the day. Nailed the structure-exploiting core that usually trips him up; the remaining gap is pushing through optimization walls instead of stopping at "not sure."

**Time Taken: 35 minutes**
