# DSA Round Transcript
**Date:** 2026-08-19
**Start Time:** 13:56:29 · **End Time:** 15:08:04 · **Duration:** 70 min (round content complete at 15:06:27)
**Problem:** Minimum Cost to Make Array Equal
**Topic:** Greedy / weighted median on a convex piecewise-linear cost function (prefix-sum implementation)
**Difficulty:** Medium-Hard
**Performance Rating:** 3/5  <!-- machine-read on future rounds; ≤2 = eligible for re-ask, ≥3 retired -->
**Hints Used:** 0/2  (requested one at +5m; declined, solved unaided)
**Constraints Asked:** `n`, value range of `nums[i]`, range of `cost[i]` — one question, up front, before writing anything · **Never Asked:** duplicates in `nums`, `n == 1`, and never returned to `nums[i] <= 10^6` as a lever
**Self-Verified:** Partially — gave the right answer (3) for the named input, but by re-deriving the problem's answer rather than executing his own indices. Claimed output was correct.
**Would it have fit a real 45-min round?** No — cut off **during the approach phase at minute 45, with zero code written.** Approach landed at minute 43.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Clarify | 4 min | 1 min | −3 | On pace |
| Approach + dry run | 15 min | 43 min | **+28** | Over |
| Code complete | 35 min | 65 min | **+30** | Over |
| Test + complexity | 42 min | 70 min | **+28** | Over |
| **Total** | 45 min | **70 min** | **+25** | Over |

Stamps: start 13:56:29 · constraints asked/answered 13:57:35 (+1m) · brute force stated 14:01:10 (+5m) · brute-force complexity restated 14:01:44 (+5m) · hint requested & declined 14:01:44 (+5m) · **approach settled 14:39:17 (+43m)** · boilerplate 14:39:40 (+43m) · code submitted 15:01:04 (+65m) · dry run 15:03:55 (+67m) · complexity corrected 15:06:10 (+70m) · optimization question 15:06:27 (+70m) · feedback 15:08:04

**The 38-minute silence between +5m and +43m consumed 54% of the round.**

---

## Problem Statement

You are given two arrays `nums` and `cost`, of the same length.

You can perform the following operation any number of times:
- Increase **or** decrease **any** element of `nums` by `1`.

The cost of increasing or decreasing `nums[i]` by `1` is `cost[i]`.

Return the **minimum total cost** such that all the elements of `nums` become **equal**.

**Example 1:**
```
Input:  nums = [1,3,5,2], cost = [2,3,1,14]
Output: 8
Explanation: Make all elements equal to 2.
  |1-2| * 2  = 2
  |3-2| * 3  = 3
  |5-2| * 1  = 3
  |2-2| * 14 = 0
  Total = 8, which is the minimum possible.
```

**Example 2:**
```
Input:  nums = [2,2,2,2,2], cost = [4,2,8,1,3]
Output: 0
Explanation: All elements are already equal, so no operations are needed.
```

Constraints were withheld and released on request (see Conversation Log).

---

## Conversation Log

**Interviewer:** [presents problem, Medium-Hard, reference timeline, clock measured not enforced] Any clarifying questions?

**Aayush:** what are the constraints on inputs

**Interviewer:** `n == nums.length == cost.length`; `1 <= n <= 10^5`; `1 <= nums[i], cost[i] <= 10^6`

**Aayush:** at the end each element of array will be equal to lets say x and since we want to minimise the cost x must be in [min nums[i], max nums[i]].
So a brute force approach is to iterate over each element in the range and calculate the cost to convert entire array to that number and take minimum of all.
This will take O(n^2) complexity.

**Interviewer:** You said "iterate over each element in the range" and called it O(n²). Which range are you iterating over, and how does that give n²?

**Aayush:** it is O(n*max(nums[i]-min nums))

**Interviewer:** Right. What's your approach?

**Aayush:** give me a hint

**Interviewer:** Not yet — take a real swing at it first. What have you got, even if it's wrong?

**Aayush:** *[38 minutes elapse]* the final value must be an element in the array only. Now we can make a pair of {nums[i], cost[i]} and sort it according to nums[i]. So finally the final state of all elements will be some index j in the new pair array. Let this index be j.
now the total cost for making final state nums[j] becomes
total cost = nums[j]*(sum of cost(0:j-1) - sum of cost(j:n-1)) + (sum of cost·nums(0:j-1) - sum of cost·nums(j:n-1))
So we need to find the j where the cost will be minimum; we can have prefix and suffix sum arrays for cost and cost·nums arrays. This will allow to get cost at index i in O(1)

*(Note: the second bracket's sign is inverted as written — the correct form is suffix minus prefix. He typed it correctly in the code.)*

**Interviewer:** Go ahead and code it.

**Aayush:** give boilerplate code

**Interviewer:** [provides `Solution` class skeleton + `main` driver carrying only the two statement examples]

**Aayush:** [submits solution — see Solution section] TC is O(n) and SC is O(n)

**Interviewer:** Dry-run your code by hand on `nums = [1,4], cost = [5,1]` and tell me what it returns.

**Aayush:** answer is 3 change everything to 1

**Interviewer:** You said O(n) time. Walk me through where that comes from.

**Aayush:** O(nlogn) time to include the sorting, n for getting the answer

**Interviewer:** Can it be done better than O(n log n)?

**Aayush:** no we need the sorted order of the nums

**Interviewer:** That's the round. [feedback follows]

---

## Solution

**Aayush's Final Solution:**
```cpp
long long minCost(vector<int>& nums, vector<int>& cost) {
    int n = nums.size();
    vector<pair<long long, long long>> v(n);
    for (int i = 0; i < n; i++) v[i] = {nums[i], cost[i]};
    sort(v.begin(), v.end());

    // prefC[i] = sum of cost in [0,i], prefW[i] = sum of cost*num in [0,i]
    vector<long long> prefC(n + 1, 0), prefW(n + 1, 0);
    vector<long long> sufC(n + 2, 0), sufW(n + 2, 0);

    for (int i = 0; i < n; i++) {
        prefC[i + 1] = prefC[i] + v[i].second;
        prefW[i + 1] = prefW[i] + v[i].second * v[i].first;
    }
    for (int i = n - 1; i >= 0; i--) {
        sufC[i] = sufC[i + 1] + v[i].second;
        sufW[i] = sufW[i + 1] + v[i].second * v[i].first;
    }

    long long ans = LLONG_MAX;
    for (int i = 0; i < n; i++) {
        long long x = v[i].first;
        long long left  = x * prefC[i] - prefW[i];        // j < i
        long long right = sufW[i + 1] - x * sufC[i + 1];  // j > i
        ans = min(ans, left + right);
    }
    return ans;
}
```
**Verified by interviewer: CORRECT.** Both statement examples, the named dry-run input, and in general. `prefC[i]` correctly excludes index `i`; `sufC[i+1]` correctly excludes it too; the element at `i` contributes zero; sentinel zeros in the `n+1`/`n+2`-sized arrays handle `i=0` and `i=n-1` with no special case. Overflow sized correctly — `x * prefC[i]` peaks near 10^17, and he made the pair `<long long, long long>` rather than `<int,int>`.

**Notable:** the sign inversion present in his written formula (prefix−suffix on the moment term) did **not** reach the code — he typed the correct suffix−prefix form. Caught silently during implementation.

**Optimal Solution (better than what he reached):**
```cpp
// O(n log n) as written; O(n) expected with nth_element instead of sort.
long long minCost(vector<int>& nums, vector<int>& cost) {
    int n = nums.size();
    vector<pair<long long,long long>> v(n);
    long long total = 0;
    for (int i = 0; i < n; i++) { v[i] = {nums[i], cost[i]}; total += cost[i]; }
    sort(v.begin(), v.end());
    long long acc = 0, target = -1;
    for (int i = 0; i < n; i++) {           // weighted median
        acc += v[i].second;
        if (2 * acc >= total) { target = v[i].first; break; }
    }
    long long ans = 0;
    for (int i = 0; i < n; i++) ans += llabs((long long)nums[i] - target) * cost[i];
    return ans;
}
```
Also available and never considered, because the value bound went unspent:
- **Counting-bucket by value** — `nums[i] <= 10^6` means no comparison sort is needed: `O(n + maxV)`.
- **Ternary search on `x`** over `[1, 10^6]`, licensed by convexity: `O(n log maxV)` ≈ 20n, eight lines, no sort.

**Time Complexity:** his answer O(n) on submission — **wrong**; corrected to O(n log n) when asked to justify · **Space Complexity:** O(n), correct

---

## Feedback Given

### Round conditions
- **Hints used: 0/2.** He asked for one at minute 5 and was sent back to work; everything after was his. No ceiling applies.
- **Constraints asked:** `n`, value range of `nums[i]`, range of `cost[i]` — one question, up front, before writing anything. Correct instinct, correctly timed.
- **Never asked:** duplicates in `nums`, `n == 1`, and — the one that mattered — never returned to `nums[i] <= 10^6` as a *bounded value range*, the constraint that unlocks sub-O(n log n).
- **Self-verified:** Partially. Right answer for `[1,4]/[5,1]` (3), but reached by reasoning about the problem, not by executing `prefC`/`sufW`/`left`/`right`. Third round running where the "trace" is a re-derivation rather than an execution. They agreed here; they won't always.
- **Code correctness:** verified silently — **correct**, first submission, no bug. And the sign error in his written formula did not reach the code.

### Rubric

**Problem understanding & clarification — 4/5.** One question, asked first, covering all three bounds that matter. That's the shape this should always take. Half a mark off for never returning to those numbers: `nums[i] <= 10^6` was in hand from minute 1 and treated as decoration rather than a lever.

**Approach & thought process — 5/5**, and it needs saying plainly: **he derived a correct, essentially-optimal solution with zero help on a problem where most candidates flail.** Brute force over the value range → optimum sits at one of the array's own values → sort → cost splits into a left-moves-up term and a right-moves-down term → prefix/suffix sums make each candidate O(1). Every step real. He also correctly restated the brute force as `O(n × range)` when pushed, rather than defending the sloppy "O(n²)".

**Code quality & correctness — 5/5.** Correct on first submission, genuinely clean. Separate `prefC`/`prefW`/`sufC`/`sufW` with the off-by-one handled by sizing `n+1`/`n+2` and letting sentinel zeros do the boundary work. The comment naming what `prefC[i]` and `prefW[i]` mean is why the indices came out right — first one he's written in a while.

**Complexity analysis — 2/5.** Submitted "TC is O(n)" directly above code whose second statement is `sort(...)`. Corrected immediately when asked to walk through it, so the knowledge is there — he just didn't look at his own code before stating it. Same failure mode as not tracing: the claim is produced from memory of intent rather than from the artifact. Space O(n), correct.

**Communication — 3/5.** Clear when speaking. But minute 5 to minute 43 was total silence. Thirty-eight minutes of silence in a 45-minute round is not a thinking style, it's an information blackout — the interviewer can't tell if he's two steps away or lost, and gets no chance to unblock him cheaply. He re-emerged with the complete solution, so the thinking was good. Nobody watching would have known.

**Time management — 1/5.**

### Pace report
[table as above]

**Would this have fit a real 45-minute round? No — and not narrowly.** Cut off **during the approach phase at minute 45, with no code written at all.** Approach landed at minute 43; two minutes left. The flawless implementation would never have been seen. On paper: *asked good clarifying questions, described a correct approach at the buzzer, wrote nothing.* A no-hire on a problem he actually solved.

**Biggest time sink, unambiguous: the 38-minute block from minute 5 to minute 43.** Everything else was fine — clarification took one minute, coding 22 minutes for 25 lines of index-sensitive code, reasonable. One silent stretch consumed 54% of the round.

Two process changes:
1. **Think out loud at the point he asked for the hint.** At minute 5 he had the brute force and its complexity — a perfectly good place to be. What should have followed is narration: *"O(n × 10^6) is too slow. The range is 10^6 but there are only 10^5 values, so most of it is wasted. Can I argue the optimum is at one of the array values?"* Those are the first three steps of the derivation he eventually did in silence. Narrating costs nothing and makes 38 minutes read as progress instead of absence.
2. **A wrong-but-reasoned attempt beats a hint request, and beats silence by more.** He asked at minute 5, was declined, and solved it unaided — so the hint wasn't needed, the *reflex* was. In a real round the interviewer gives it, he takes it, and he finishes with a capped score on a problem he could do cold.

### **Performance Rating: 3/5**

No ceiling binds — zero hints, no uncaught bug, constraints asked unprompted. On the merits: **an optimal solution, derived unaided, implemented correctly first try, delivered 25 minutes after the interview ended.** Solution quality alone is 5-shaped. Pace at 70 minutes and a complexity claim contradicted by the line above it pull it to 3. Nothing here was a knowledge gap.

---

## Algorithmic Thought-Process Debrief

**Topic tag:** greedy / weighted median on a convex piecewise-linear cost function (prefix-sum implementation).

### 1. The derivation chain

**Trigger:** the answer is a single unknown value `x`, and the cost is a function of it.
**Move:** write the objective explicitly — `f(x) = Σ cost[i] · |nums[i] − x|` — and stop treating it as an array problem. It's a 1-D minimisation. He did this implicitly; writing it down makes the next three steps mechanical.

**Trigger:** `f` is a sum of absolute values, each scaled by a positive weight.
**Move:** each term `cost[i]·|nums[i] − x|` is a V — convex, piecewise linear, kink at `nums[i]`. Sum of convex is convex; sum of piecewise-linear is piecewise linear with kinks only at the union of individual kinks. **Therefore the minimum is attained at some `nums[i]`.** He asserted this and moved on; it's the load-bearing claim of the whole solution and has a two-line proof.

**Trigger:** n candidates × O(n) each → O(n²).
**Move:** name the redundant work. Moving the candidate from `nums[j]` to `nums[j+1]` doesn't change *which* elements are left or right — only `x` changes. So cost decomposes into an `x`-dependent and an `x`-independent part per side:
```
f(x) = x·(Σ_{i<j} c_i) − (Σ_{i<j} c_i·n_i) + (Σ_{i>j} c_i·n_i) − x·(Σ_{i>j} c_i)
       └ weight left ┘  └ moment left ─┘    └ moment right ─┘  └ weight right ┘
```
Four prefix/suffix sums, O(1) per candidate → **O(n log n)**, dominated by the sort. This is where he landed, and it's a respectable final answer.

**Trigger (the step past his stopping point):** slide `x` right by one unit. Everything left gets more expensive by `Σ_{i≤x} cost[i]`; everything right gets cheaper by `Σ_{i>x} cost[i]`. So the slope is `leftWeight − rightWeight`, **monotonically increasing** — negative, then positive, crossing zero exactly once.
**Move:** don't evaluate all n candidates. Find the first `j` where the slope flips: smallest `j` with `prefC[j+1] ≥ totalCost / 2`. That's the **weighted median** — one pass, and since only a selection is needed, `nth_element` gives **O(n) expected**, no sort.

**And the constraint never spent:** `nums[i] <= 10^6`. A bounded value range means no comparison sort was ever required — bucket weights by value and sweep, **O(n + maxV)**. Or, since convexity is established, **ternary search on `x`** over `[1, 10^6]`: `O(n log maxV)` ≈ 20n, eight lines.

### 2. The signal he missed

He had it and put it down. When he wrote
> `total cost = nums[j]*(sum of cost(0:j-1) - sum of cost(j:n-1)) + ...`

that first bracket — `leftCostSum − rightCostSum` — **is the derivative of `f` at `x`**. He wrote the slope down, used it as a coefficient, and never asked what it does as `j` increases. It increases monotonically, because every step moves one element's weight from the right sum to the left sum. One monotone quantity crossing zero once is a binary search, a single scan, or a selection — never an exhaustive evaluation of every candidate.

Not a missing insight — an insight he derived, used mechanically, and didn't interrogate.

### 3. The generalization

**The class:** *minimise `Σ w_i · |a_i − x|` over `x`* — and everything reducible to it. Equalise an array at minimum cost, meeting-point / minimum-total-travel, best pivot on a line, minimum moves to make all elements equal, optimal warehouse placement on a road.

**The tell:** the objective is a **sum of absolute values**. Reflexes:
- `Σ |a_i − x|` (unweighted) → **median**.
- `Σ w_i·|a_i − x|` → **weighted median** — smallest prefix where cumulative weight ≥ half the total.
- `Σ (a_i − x)²` → **mean**. (Absolute → median, squared → mean; the contrast is worth holding.)
- Any of the above with a constraint on `x` → convex, so **clamp** the unconstrained optimum to the feasible range, or ternary-search it.

The transferable reflex: **when the cost function is a sum of convex pieces, stop enumerating candidates and start looking at the slope.** Convexity converts "try every option" into "find where the derivative changes sign" — a scan, a binary search, or a selection.

Meta-tell that cost him the better solution: he finished with a bounded-value constraint (`10^6`) unspent. Before settling on a complexity, run the check — *which given constraint have I not used?* An unused bound is almost always an invitation.

### 4. Drill

**(a) The technique — 25 minutes, four problems, weighted-median framing forced.** For each, before coding, write `f(x) = Σ ___ · |___ − x|` and name the answer as median / weighted median / mean *before* writing any loop:
1. **LC 462 — Minimum Moves to Equal Array Elements II** (unweighted; median)
2. **LC 2448 — this problem**, re-solved via weighted median in one pass, no prefix arrays
3. **LC 296 — Best Meeting Point** (2-D, separates into two independent 1-D medians — the separation is the insight)
4. **LC 1478 — Allocate Mailboxes** (weighted median as the inner cost of a DP — the pattern as a subroutine)

**(b) The pace — the one that decides his next round.** Next three problems under a **hard 12-minute approach timer, narrating continuously.** Not thinking silently and reporting at the end — talking the whole way, dead ends included. Rule: **no more than 90 seconds of silence.** If stuck, say what he's stuck on out loud; that sentence usually unsticks it, and in a real interview it's the only thing standing between him and a no-hire on a problem he can solve.

His solving is not the problem. Today he produced an optimal, correct, cleanly-implemented solution to a hard problem with no help whatsoever — 25 minutes after the interview ended, in silence, having asked for a hint he didn't need. Fix the clock and the narration and this is a 5.
