# DSA Round Transcript
**Date:** 2026-06-12
**Start Time:** 10:09
**End Time:** 10:51
**Duration:** 42 minutes
**Problem:** Maximum Sum Circular Subarray
**Topic:** Arrays / Kadane's Algorithm
**Difficulty:** Medium

---

## Problem Statement
Given a **circular** integer array `nums` of length `n`, return the maximum possible sum of a **non-empty subarray** of `nums`.

A circular array means the end connects to the beginning: the next element of `nums[i]` is `nums[(i + 1) % n]`, and the previous is `nums[(i - 1 + n) % n]`. A subarray may include each element of the fixed buffer at most once (no double-counting on wrap).

**Example 1:** `nums = [1, -2, 3, -2]` → `3` (subarray `[3]`)
**Example 2:** `nums = [5, -3, 5]` → `10` (subarray `[5, 5]` wrapping around)
**Example 3:** `nums = [-3, -2, -3]` → `-2` (subarray `[-2]`)

**Constraints:**
- `1 <= n <= 3 * 10^4`
- `-3 * 10^4 <= nums[i] <= 3 * 10^4`

---

## Conversation Log

**Interviewer:** Presented the problem and asked for clarifying questions / approach.

**Aayush:** What are the constraints on the elements of `nums` and the size?

**Interviewer:** `1 <= n <= 3*10^4`, `-3*10^4 <= nums[i] <= 3*10^4`. Non-empty, can be negative/zero/positive, fits in 32-bit int.

**Aayush:** If subarrays were simple (non-wrapping) we could use the maximal sum subarray algorithm (Kadane). But when subarrays can wrap around, we need to also account for that case.

**Interviewer:** Right — Kadane for the non-wrap case. How do you handle wrapping?

**Aayush:** The wrapping case would be `total sum - minimal sum subarray` (min subarray via Kadane).

**Interviewer:** Elegant. Answer is `max(kadaneMax, total - kadaneMin)`. Code it up — and think about whether that formula is always safe.

**Aayush:** (Wrote initial O(n)-space C++ solution computing `kadaneMax`, `kadaneMin`, `total`, output `max(kadaneMax, total - kadaneMin)`. Claimed O(n) time, O(n) space.)

**Interviewer:** Before I accept it — hand-trace Example 3, `nums = [-3, -2, -3]`. What does it print vs. what should it print?

**Aayush:** `total = kadaneMin = -8`, `kadaneMax = -2`, but answer comes out `0`. Need to handle the all-negative case where `total == kadaneMin`. Fix: `if (total == kadaneMin) ans = kadaneMax;`

**Interviewer:** Your fix produces right answers, but your comment says `total == kadaneMin` only when the whole array is negative. Is that true? Trace `[0, -2, 0]`.

**Aayush:** No, not all negative. (`total = kadaneMin = -2` but array has zeros.)

**Interviewer:** So the stated reason is wrong. Does the code still output correctly on `[0,-2,0]`, and what's the *real* reason the fix is safe?

**Aayush:** When `total == kadaneMin`, the maximal wraparound candidate is `0`, so... `kadaneMax >= 0` if at least one element is non-negative.

**Interviewer:** Exactly — in the non-all-negative case `kadaneMax >= 0` already captures that `0`, so discarding the wrap candidate loses nothing; in the all-negative case the `0` is an invalid empty subarray and must be discarded. Returning `kadaneMax` is safe in both branches. Now — can you reduce the O(n) space?

**Aayush:** Keep running max/min variables since we only need the last value — brings space to O(1).

**Interviewer:** Right — write the tightened version.

**Aayush:** (Wrote O(1)-space version with `curMax`/`curMin` scalars; fix intact.)

**Interviewer:** Correct and clean. (Nit: the all-negative comment is now stale given the real reasoning.) Done — note the time.

---

## Solution
**Aayush's Final Solution (O(1) space):**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
  vector<int> nums{-5, -3, -5};

  int n = nums.size();
  int total = accumulate(nums.begin(), nums.end(), 0);

  int kadaneMax = nums[0], curMax = nums[0];
  for (int i = 1; i < n; i++) {
      curMax = max(nums[i], curMax + nums[i]);
      kadaneMax = max(curMax, kadaneMax);
  }

  int curMin = nums[0], kadaneMin = nums[0];
  for (int i = 1; i < n; i++) {
      curMin = min(nums[i], curMin + nums[i]);
      kadaneMin = min(kadaneMin, curMin);
  }

  int ans = max(kadaneMax, total - kadaneMin);
  if (total == kadaneMin) {  // wrap candidate is empty/0; kadaneMax already covers it
      ans = kadaneMax;
  }
  cout << "ans is " << ans;
  return 0;
}
```

**Optimal Solution:** Same as Aayush's — this is the optimal approach.

**Time Complexity:** O(n)
**Space Complexity:** O(1)

---

## Feedback Given

This was one of your stronger sessions. You hit the core insight fast and your correctness reasoning held up under pressure.

**What went well:**
- Asked for constraints upfront before approaching — exactly the clarification habit you've historically skipped.
- Structure-exploiting approach immediately: split into non-wrap (Kadane) vs wrap, and produced the `total - minSubarray` formula unprompted.
- Repaired your own reasoning under questioning: your first justification for the `total == kadaneMin` fix was wrong, you owned it on `[0,-2,0]`, then reconstructed the correct argument (`kadaneMax >= 0` when any element is non-negative).
- Self-driven space optimization O(n) → O(1) with a clean, correct rewrite.

**What to sharpen:**
- Didn't self-catch the all-negative edge case despite an explicit "think hard about whether that formula is always safe" prompt before coding. It only surfaced when forced to hand-trace Example 3 (which was in the problem statement). Build the reflex: after writing a formula, feed it the adversarial input yourself.
- Correct code ≠ understood code. Shipped a fix with a comment stating a false reason. Lead with the precise *why*, not a plausible-sounding one.

**Scoring (1–5):**
| Dimension | Score | Notes |
|---|---|---|
| Problem understanding & clarification | 4.5 | Asked constraints proactively — real improvement |
| Approach & thought process | 4.5 | Nailed the structural insight unprompted |
| Code quality & correctness | 4 | Clean, but edge case only found via dry-run prompt; stale comment |
| Complexity analysis | 5 | Correct, and optimized space yourself |
| Communication | 4 | Conceded and rebuilt reasoning well; initial justification imprecise |

**Overall: ~4.4/5.** A genuinely senior-level performance. The one habit standing between you and a clean pass: verify your own formula against the nasty input before saying "done."

**Time Taken: 42 minutes**
