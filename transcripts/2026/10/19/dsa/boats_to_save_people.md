# DSA Round Transcript
**Date:** 2026-10-19
**Start Time:** 18:25:09 · **End Time:** 18:45:57 · **Duration:** 21 min
**Problem:** Boats to Save People (LC 881)
**Topic:** Greedy + two pointers on sorted array (heaviest pairs with lightest; exchange argument)
**Difficulty:** Medium
**Performance Rating:** 4/5  <!-- machine-read on future rounds; ≤2 = eligible for re-ask, ≥3 retired -->
**Hints Used:** 0/2
**Constraints Asked:** generic "what are the constraints?" (got n ≤ 5·10⁴, 1 ≤ people[i] ≤ limit ≤ 3·10⁴) · **Never Asked:** nothing material left open — but never converted the bounds into a budget, and never used the bounded weight range
**Self-Verified:** No — submitted without tracing. When asked, traced [1,1,2,2], limit 3 → 2 (correct).
**Would it have fit a real 45-min round?** Yes — done at +21 with room to spare.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Clarify | 3 min | +4m (18:29:32 — first approach arrived; question itself answered at +1) | +1 | Over by 1 min |
| Approach + dry run | 12 min | +15m (18:40:45) | +3 | Over by 3 min |
| Code complete | 30 min | +19m (18:44:15) | -11 | On pace |
| Test + complexity | 40 min | +21m (18:45:57) | -19 | On pace |
| **Total** | 45 min | 21 min | -24 | Fit |

---

## Problem Statement
`people[i]` is the weight of person i; `limit` is the max weight per boat; unlimited boats. Each boat carries **at most two people**, provided their weight sum is **at most `limit`**. Return the minimum number of boats to carry everyone.

Example 1: `people = [3,2,2,1], limit = 3` → 3 (boats (1,2), (2), (3))
Example 2: `people = [3,5,3,4], limit = 5` → 4 (boats (3), (3), (4), (5))

---

## Conversation Log
**Interviewer [18:25:09 · +0m]:** Presented problem, difficulty, reference timeline (measured, not enforced). Any clarifying questions?
**Aayush:** what are the constraints?
**Interviewer [18:26:25 · +1m]:** 1 ≤ n ≤ 5·10⁴; 1 ≤ people[i] ≤ limit ≤ 3·10⁴. What's your approach?
**Aayush:** Sort, then keep putting people[i] into the current boat; if it already has 2 people or the weight would exceed limit, start a new boat.
**Interviewer [18:29:32 · +4m]:** Run that on [1,1,2,2], limit 3. Which boats, how many?
**Aayush:** It would build 3 boats but min is 2.
**Interviewer [18:29:56 · +4m]:** Right — so what's your approach now?
**Aayush:** Sort; two pointers lo = 0, hi = n−1. If p[lo] + p[hi] ≤ limit, both go in one boat; else only hi goes, hi−−. Repeat while lo ≤ hi.
**Interviewer [18:38:26 · +13m]:** Your first greedy looked reasonable and was wrong — why is this one right? (1) When p[lo]+p[hi] > limit, why is sending hi alone safe? (2) When they fit, why pair hi with the lightest rather than a heavier person who'd also fit?
**Aayush:** (1) We want two per boat where possible; with current lo and hi the best choice is to reduce hi in hopes lo and hi−1 fit. (2) So the heavier person can pair with another hi' < hi.
**Interviewer [18:39:43 · +14m]:** On 1 — "in hopes" isn't a proof. Can sending hi alone at that moment ever cost a boat versus optimal? Yes or no, and why.
**Aayush:** Safe — hi can't pair with any index > lo since those are heavier, so hi goes alone.
**Interviewer [18:40:45 · +15m]:** Okay. Go ahead and code it.
**Aayush:** cpp boilerplate code
**Interviewer [18:40:53 · +15m]:** Provided boilerplate with the two statement examples.
**Aayush:** Submitted code (below). TC O(n log n), SC O(1), "this is the most optimal code".
**Interviewer [18:44:15 · +19m]:** Dry-run on [1,1,2,2], limit 3 — lo, hi, ans per iteration and return value. Justify "most optimal" for time and space.
**Aayush:** (lo,hi,ans): (0,3,0) → (1,2,1) → (2,1,2); returns 2. n log n for sorting, two-pointer pass is O(n).
**Interviewer [18:45:57 · +21m]:** Feedback (below).

---

## Solution
**Aayush's Final Solution:**
```cpp
int numRescueBoats(vector<int>& people, int limit) {
    sort(people.begin(), people.end());
    int ans = 0;
    int lo = 0, hi = people.size() - 1;
    while (lo <= hi) {
        if (people[hi] + people[lo] > limit) hi--;
        else { lo++; hi--; }
        ans++;
    }
    return ans;
}
```
Verified correct on both examples, [1,1,2,2]/3 → 2, single-element and lo == hi cases.

**Optimal Solution (exploits the weight bound — counting sort, O(n + limit)):**
```cpp
int numRescueBoats(vector<int>& people, int limit) {
    vector<int> cnt(limit + 1, 0);
    for (int w : people) cnt[w]++;
    int lo = 1, hi = limit, ans = 0;
    while (true) {
        while (lo <= limit && cnt[lo] == 0) lo++;
        while (hi >= 1 && cnt[hi] == 0) hi--;
        if (hi < 1) break;                       // everyone placed
        cnt[hi]--;                               // heaviest always boards
        if (lo <= hi && cnt[lo] > 0 && lo + hi <= limit) cnt[lo]--;   // lightest joins if it fits
        ans++;
    }
    return ans;
}
```
**Time Complexity:** O(n log n) (his) / O(n + limit) (counting) · **Space Complexity:** he said O(1) — actually O(log n) sort stack (introsort); counting version O(limit).

---

## Feedback Given

### Round conditions
- **Hints used: 0/2.** `[1,1,2,2]` was a failing input, which isn't a hint, and everything else was "justify that". No hint ceiling applies.
- **Constraints:** you asked the generic question first, which was good. But you never turned n ≤ 5·10⁴ into a target, and you never noticed that **weights are bounded by 3·10⁴**. That bound is the one fact you didn't use.
- **Self-verification:** you submitted without a trace and called it "most optimal". When asked, your trace of `[1,1,2,2]` was correct and well laid out (lo/hi/ans per step), and it returned 2.

### Rubric
- **Problem understanding & clarification — 3/5.** You asked for constraints right away. You didn't use either bound: one should have set the target complexity, the other opens up a counting sort.
- **Approach & thought process — 4/5.** Your first greedy (fill boats one after another in sorted order) came out in 3 minutes with no argument for why it's optimal, and one 4-element input broke it. You recovered to two pointers without help. When pushed, your proof of decision 1 was correct and crisp: "lo is the lightest left, so if hi doesn't fit with lo it fits with nobody." Decision 2 was hand-wavy. The full exchange argument: in any optimal answer, if hi is paired with some x, swapping x and lo still fits, because lo ≤ x. Lead with that before you code.
- **Code quality & correctness — 5/5.** Clean and correct, and handles `lo == hi` without a special case.
- **Complexity analysis — 3/5.** O(n log n) time is right for your code. "O(1) space" ignores the sort's O(log n) stack. "Most optimal" was a claim you didn't check: with weights capped at 3·10⁴, counting sort makes it O(n + limit).
- **Communication — 4/5.** The trace was the best of today's three rounds: per-step state, no bare numbers. The weak spots were "in hopes" as a proof, and announcing "most optimal" without being asked.
- **Time management — 5/5.** See the pace report.

### Pace report
| Phase | Reference | Actual | Verdict |
|---|---|---|---|
| Clarify | 3 | 4 | over by 1 (the 3 min after your question went into the first approach) |
| Approach + dry run | 12 | 15 | over by 3 |
| Code complete | 30 | 19 | on pace |
| Test + complexity | 40 | 21 | on pace |
| Total | 45 | 21 | fit, with 24 min to spare |

**Would this have fit a real 45-minute round? Yes, easily.** The only sink was 18:29 → 18:38: 8.5 minutes to get from the failed greedy to two pointers.

**Performance Rating: 4/5.** You used no hints, the code is correct, and you were well under time. It's not a 5 because the first approach had no proof, the "most optimal" claim went unchecked, and the bounded-weight constraint went unused. No hard ceiling applied.

### Algorithmic Thought-Process Debrief
1. **The derivation chain**
   - **Brute force:** try every pairing. The real question is who the heaviest person rides with.
   - **Look at the heaviest remaining person, `hi`:** if even the lightest person can't ride with `hi`, nobody can, so `hi` goes alone. Nothing is lost.
   - **If the lightest fits with `hi`:** pairing them is never worse. In any optimal answer, swapping `hi`'s partner for the lightest still fits (exchange argument).
   - **Run it:** each step removes `hi` and possibly `lo`. That's two pointers moving inward on a sorted array.
   - **Spend the last constraint:** weights ≤ 3·10⁴, so a counting sort replaces the comparison sort and gives O(n + limit).
2. **The signal you missed:** your first greedy used sorted order without asking which person is hardest to place. That's the heaviest person, and every decision should be made from their side. Later, "most optimal" skipped the bounded-weights fact, which was sitting in the constraints you asked for.
3. **The generalization:** "pair items under a capacity, at most 2 per group" → sort, then match the heaviest with the lightest using two pointers, justified by an exchange argument. Same family: Two Sum ≤ target counting, Minimum Number of Arrows, Assign Cookies, Max Number of K-Sum Pairs. **Tell:** a pairwise compatibility test that's monotone in weight. Separate trigger: a small value range means counting sort or bucketing beats comparison sort.
4. **Drill:** before you state any greedy, say its exchange argument in one sentence: "take any optimal solution; if it does X differently, swap to my choice and it stays feasible and no worse." If you can't finish the sentence, build a 4-element counterexample before saying the greedy out loud. Also, at "can it be optimized?", read the constraints again and name one you haven't used.
