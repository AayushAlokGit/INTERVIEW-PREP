# DSA Round Transcript
**Date:** 2026-08-13
**Start Time:** 21:28:16 · **End Time:** 22:46:00 · **Duration:** 78 min
**Problem:** Longest Well-Performing Interval
**Topic:** Prefix sums + monotonic stack / first-occurrence hashmap
**Difficulty:** Medium-Hard
**Performance Rating:** 2/5  <!-- machine-read on future rounds; <=3 = eligible for re-ask, >=4 retired -->
**Hints Used:** 0/2
**Constraints Asked:** one generic "what are the constraints" (got n <= 10^4, hours[i] in [0,16]) · **Never Asked:** strictness of "> 8", empty-answer convention, whether interval must be non-empty, empty-array handling
**Self-Verified:** Partially — revised his own code once catching two off-by-ones, then claimed output 3 on the example. Claimed output was correct for that input, but the submitted code is wrong in general (returns 1 on [6,6,9,9], answer 3).
**Would it have fit a real 45-min round?** No — cut off during the approach phase (36-min silent block); would have reached no submitted solution at 45 min.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Clarify | 4 min | 2 min | −2 | Yes |
| Approach + dry run | 15 min | 43 min | +28 | No |
| Code complete | 35 min | 75 min | +40 | No |
| Test + complexity | 42 min | 77 min | +35 | No |
| **Total** | 45 min | 78 min | +33 | No |

---

## Problem Statement

You're given an array `hours`, where `hours[i]` is the number of hours an employee worked on day `i`.

A day is considered a **tiring day** if the employee worked **more than 8 hours** on that day.

A **well-performing interval** is a contiguous interval of days in which the number of tiring days is **strictly greater** than the number of non-tiring days.

Return the **length of the longest well-performing interval**.

**Example 1**
```
Input:  hours = [9, 9, 6, 0, 6, 6, 9]
Output: 3
Explanation: The longest well-performing interval is [9, 9, 6].
```

**Example 2**
```
Input:  hours = [6, 6, 6]
Output: 0
```

(Constraints withheld from the statement; supplied on request.)

---

## Conversation Log

**Interviewer [21:28:16 · +0m]:** Presented the problem. Difficulty Medium-Hard, reference timeline stated as measured but not enforced. Asked for clarifying questions.

**Aayush [21:30:32 · +2m]:** "wht are th econstraints ?"

**Interviewer [21:30:32 · +2m]:** `1 <= hours.length <= 10^4`, `0 <= hours[i] <= 16`. Anything else?

**Aayush [21:33:34 · +5m]:** "the conditions boils down to length of longest [i:j] where 2*T > length of [i:j] where T - no of tiring days in [i:j]. A brute force solution can be of O(n^2) with a prefix array count of tired days."

**Interviewer [21:33:34 · +5m]:** That's a correct brute force. Where do you want to go from there?

**Aayush [22:09:07 · +41m]:** "if we convert all days to 0 or 1 depending on whether they are tiring or not. We can get prefix[i] to get number of tiring day in [0:i]. Now we need to find maximum length interval [i:j] satisfying 2*(p[j] - p[i-1]) > j-i+1 (i>=1) or 2*p[j] > j if i=0. This boils down to 2*p[j] - j > 2(p[i-1]) - (i-1). Basically so we need the indices in monotonically increasing order for the function 2*p[i]-i. If we are at an index we need to find the last index i which satisfied f[i] < f[j] so we can use a monotonic stack to ensure monotonic order for the f[i] — the stack will just contain the indices."

**Interviewer [22:09:07 · +41m]:** Two things I want you to state precisely before you code: what exactly does the stack hold at the end, and when you're at `j`, which qualifying `i` do you want — and why that one?

**Aayush [22:10:45 · +43m]:** "the stack holds at the end the indices in increasing order of 2*p[i] - i. So st.top() will have the index with max 2*p[i] - i. All indices before it would have 2*p[index]-index < the value for st.top() so they can qualify a potential candidates for the left boundary of the window."

**Interviewer [22:10:45 · +43m]:** Alright — go ahead and code it.

**Aayush [22:10:56 · +43m]:** "give me boilerplate cpp code"

**Interviewer [22:10:56 · +43m]:** Supplied `longestWPI` signature + `main` driver carrying only the two statement examples.

**Aayush [22:41:53 · +74m]:** Submitted first version (empty-branch check `2*prefix[i] > i`, stack pop on `>`, length `i - top - 1`).

**Interviewer [22:41:53 · +74m]:** Dry-run your code by hand on `hours = [9, 9, 6, 0, 6, 6, 9]` and tell me what it returns.

**Aayush [22:43:39 · +75m]:** Submitted a revised version instead of a number — changed pop condition to `>=`, empty-branch check to `2*prefix[i] > i+1`, and length to `i - top`. No output stated.

**Interviewer [22:43:39 · +75m]:** You gave me the code but not the number. What does it return on `[9, 9, 6, 0, 6, 6, 9]`?

**Aayush [22:44:19 · +76m]:** "it returns 3 after the fixed code"

**Interviewer [22:44:19 · +76m]:** Time and space complexity?

**Aayush [22:44:51 · +77m]:** "O(n) TC - each index visited atmost twice and O(n) SC"

**Interviewer [22:44:51 · +77m]:** Can this be optimized further — either bound?

**Aayush [22:45:06 · +77m]:** "i dont think so"

**Interviewer [22:46:00 · +78m]:** Ended the round, revealed the topic tag and the bug, gave full feedback.

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int longestWPI(vector<int>& hours) {
    int n = hours.size();
    vector<int> prefix(n,0);
    prefix[0] = (hours[0]>8);
    for(int i=1;i<n;i++)
    {
        prefix[i] = prefix[i-1] + (hours[i]>8);
    }

    for(int i=0;i<n;i++)
    {
        cout<<prefix[i]<<" ";
    }
    cout<<endl;
    stack<int> st;
    int ans = 0;
    // the relevant well performing interval follows 2*p[j] - j > 2*p[i-1] - (i-1) for [i:j]

    // assuming i is the rigt boundary we try to find possible left boundaries.
    for(int i=0;i<n;i++)
    {
        cout<<"i-> "<<i<<" 2*prefix[i]-i -> "<<2*prefix[i]-i<<endl;
        // the canddiate at st top cant be a left boundary
        while(!st.empty() && 2*prefix[st.top()] - st.top() >= 2*prefix[i]-i)
        {
            st.pop();
        }
        if(st.empty() && 2*prefix[i] > i+1)
        {
            cout<<"{zero,"<<i<<"}"<<endl;
            ans = max(ans,i+1);
        }else if(!st.empty()  && 2*prefix[st.top()] - st.top() < 2*prefix[i]-i)
        {
            int leftBoundaryCandidate = st.top();
            cout<<"{"<<leftBoundaryCandidate<<","<<i<<"}"<<endl;
            ans = max(ans , i-leftBoundaryCandidate);
        }
        st.push(i);
    }
    return ans;
}
```

**Bug (verified by compiling and stress-testing against brute force):**
- Passes both statement examples (`3` and `0`).
- Fails on `[6, 6, 9, 9]` → returns **1**, correct answer **3**.
- Fails on `[6, 9, 6, 9, 6, 6, 6, 9]` → returns **1**, correct answer **3**.
- Root cause 1: his stack ends up *increasing* in `f = 2p - i`, so `st.top()` is the **latest** qualifying left boundary; he needs the **earliest** (the bottom). He verbally identified that the whole stack qualifies, then selected the element giving the shortest interval.
- Root cause 2: the `while` loop pops candidates permanently during a left-to-right pass, so an index popped at `i=2` can never serve as a left boundary at `i=6`. The structure is wrong, not just the index arithmetic.

**Optimal Solution:**
```cpp
int longestWPI(vector<int>& hours) {
    unordered_map<int,int> first;   // score -> earliest index achieving it
    int score = 0, ans = 0;
    for (int i = 0; i < (int)hours.size(); i++) {
        score += (hours[i] > 8) ? 1 : -1;
        if (score > 0) {
            ans = i + 1;                       // whole prefix works
        } else {
            auto it = first.find(score - 1);   // only score-1, never score-2
            if (it != first.end()) ans = max(ans, i - it->second);
        }
        if (!first.count(score)) first[score] = i;   // earliest only
    }
    return ans;
}
```

Key property: `score` is a ±1 lattice walk, so to reach any value `v <= score-2` it must pass through `score-1` first. Hence the earliest index with a strictly smaller score is always the earliest index with score exactly `score-1` — one hashmap lookup, no stack, no direction reversal.

Stack variant (also correct): build a **strictly decreasing** stack of prefix scores left-to-right (never popped during construction — the running prefix minima), then scan `j` right-to-left, popping while `f[stack.top()] < f[j]` and taking `j - popped`.

**Time Complexity:** his answer O(n), amortized two visits per index — correct for the algorithm he wrote · **Space Complexity:** his answer O(n) — correct.

---

## Feedback Given

### Round conditions
- **Hints used: 0/2.** No ceiling from hints — the entire derivation was his.
- **Constraints asked:** one generic "what are the constraints" at +2m → `n <= 10^4`, `hours[i] in [0,16]`. **Never asked:** strictness of "> 8", empty-answer convention, non-empty interval requirement, empty-array handling. All inferred from the examples.
- **Self-verified:** partially. Revised once, catching two off-by-ones on his own. Claimed output **3** on the example — correct for that input, but the code is wrong in general.

### Rubric
- **Problem understanding & clarification — 2.5/5.** One generic constraint ask is better than zero, but it's a reflex, not a probe. He got two numbers back and used neither. Zero semantic questions.
- **Approach & thought process — 4/5.** Genuinely strong and unaided. Brute force stated cleanly; the 0/1 transform, prefix array, and algebraic rearrangement to `2p[j] - j > 2p[i-1] - (i-1)` — the entire crux — reached with zero help. A real step up from his usual generic-template reflex. Capped only by the last mile: named the right quantity, then picked the wrong element from his own structure.
- **Code quality & correctness — 1/5.** Wrong answer on a 4-element input. Two off-by-ones self-caught, one structural bug never caught. Debug `cout`s left in the submission.
- **Complexity analysis — 4/5.** O(n)/O(n) with a correct amortized justification, stated immediately. Marked down only because it's the complexity of an algorithm that doesn't work.
- **Communication — 4/5.** Clear, compact, algebra written out. When asked to state what the stack holds, gave a precise answer rather than deflecting — a genuine improvement over past rounds.
- **Time management — 1/5.** See pace report.

### Pace report
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Clarify | 4 min | 2 min | −2 | on pace |
| Approach + dry run | 15 min | 43 min | +28 | over |
| Code complete | 35 min | 75 min | +40 | over |
| Test + complexity | 42 min | 77 min | +35 | over |
| **Total** | 45 min | 78 min | +33 | over |

**Would this have fit a real 45-minute round? No — and not close.** A real interviewer cuts him off during the **approach phase**. He went silent from 21:33 to 22:09 — **36 minutes** between "here's the brute force" and the transform. In a live round that silence ends the interview: hints start at minute 15, "needed the key insight given" is written down by minute 25, and at 45 minutes there is **no submitted solution**, only the algebra.

**Biggest time sink: the 36-minute silent block.** Not the coding (~30 min, slow but survivable). All the derivation happened where nobody could see it. He emerged with a fully-formed correct transform, so the thinking was good — but the interviewer got 36 minutes of nothing. Thinking aloud is the only thing that converts his derivation into signal.

Also named: he jumped to writing the final data structure without dry-running the *approach* on paper. Five minutes tracing `f = [2,3,2,1,0,-1,0]` against his own stack rule would have exposed the `st.top()` bug before a line of C++.

### Performance Rating: 2/5
**This would have been a 4 — optimal approach, zero hints, correct complexity — capped at 2 because he submitted code with a bug he never caught.** He shipped something broken and called it done. Pace alone would independently have made this a fail in a real round.

### Algorithmic Thought-Process Debrief

**1. The derivation chain** (he got steps 1–4 unaided; the fork is at 4→5):
1. Brute force + name the redundant work: all O(n²) intervals, O(1) each with a prefix count; the redundancy is re-scanning every left end for each right end. → Fix `j`, ask what you need from the left in O(1). ✅
2. Collapse the two-quantity condition into one: `T > (len − T)` ⟺ `2T > len`. Two counters become one. → Replace each day with ±1. ✅
3. Rearrange so each side depends on one index only: `f[k] = 2p[k] − k`, condition becomes `f[j] > f[i−1]`. → For each `j`, find the smallest `i−1` with `f[i−1] < f[j]`. ✅ (the hard step)
4. Match "smallest index with a smaller value" to a structure. Nearest-smaller-to-the-left is a monotonic stack; **earliest**-smaller-to-the-left is a different problem. ❌ fork here.
5. The missing property: `f` moves by exactly ±1, so it's a lattice walk. To reach any `v <= f[j]−2` it must pass through `f[j]−1`. The earliest index with `f < f[j]` is **always** the earliest index with `f == f[j]−1`.
6. Per-step operation: "have I seen score `f[j]−1`, and when *first*?" → hashmap keyed by score, written only on first occurrence.

**2. The signal he missed:** `f` is a ±1 walk, so it has the intermediate value property. That single fact turns "find the earliest index with any smaller value" (needs a stack plus a direction reversal) into "look up `score − 1` in a hashmap." He walked past it at 22:09:07, writing *"All indices before it would have f[index] < f[top] so they can qualify as potential candidates"* — staring directly at the qualifying set, he asked "which one does my stack hand me?" instead of "which one do I **want**, and is there a reason it's easy to find?" This is his recurring Q7: candidate set was right, **selection rule** was wrong. He optimized the wrong end.

**3. The generalization:** class = "longest subarray where a count-comparison holds," i.e. `count(A) − count(B) > k` over a contiguous range. Tell: the condition compares two counts of a binary property over a window. Move: map to ±1, take the running sum `S`; then
- equality target (`S[j] == S[i−1]`) → hashmap of first occurrence
- strict inequality **on a ±1 walk** → hashmap of first occurrence of `S[j] − 1` only, *because of ±1 continuity*
- inequality on an **arbitrary-valued** prefix → decreasing prefix-minima stack + right-to-left scan

The discriminator question is "does my prefix array move in unit steps?" He reached for the stack because he never asked. Siblings he has now seen: Subarray Sum Equals K, Find the Longest Substring Containing Vowels in Even Counts (same skeleton, rated 1/5 on 2026-08-05), Contiguous Array, Shortest Subarray with Sum at Least K.

**4. Concrete drill — the selection-rule check.** Before writing any monotonic-stack code, write one sentence as a comment: *"At index j, among all qualifying candidates in my stack, I want the ____-most one, and it is at ____."* Fill both blanks with real words. If the second blank is "the bottom" or "somewhere in the middle," the stack direction or scan direction is wrong — fix the structure, don't reach for `top()` and hope. At 22:09 that sentence would have read *"I want the earliest one, and it is at the bottom"* and caught the whole bug pre-coding.

Three problems in one sitting (~35 min), narrating aloud throughout:
1. LC 525 Contiguous Array — the equality case, warm-up
2. LC 1124 Longest Well-Performing Interval — re-derive from scratch, hashmap version
3. LC 962 Maximum Width Ramp — same "earliest qualifying index" question but with an arbitrary-valued prefix, so the hashmap trick is unavailable and the decreasing-stack + right-to-left scan is forced

Doing 2 and 3 back to back is the point: same question, two different answers, and the ±1 property is the only thing separating them.

**Hard rule for next round: no silent block longer than 90 seconds.** 36 minutes of silence turned a 4-quality derivation into a no-hire.
