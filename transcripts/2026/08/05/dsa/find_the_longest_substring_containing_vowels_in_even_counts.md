# DSA Round Transcript
**Date:** 2026-08-05
**Start Time:** 10:28:45 · **End Time:** 11:31:51 · **Duration:** 63 min
**Problem:** Find the Longest Substring Containing Vowels in Even Counts
**Topic:** Prefix state / bitmask parity + hashmap
**Difficulty:** Medium-Hard
**Performance Rating:** 1/5  <!-- machine-read on future rounds; <=3 = eligible for re-ask, >=4 retired -->
**Hints Used:** 2/2
**Constraints Asked:** one generic "what are the constraints" (received n <= 5*10^4, lowercase only) · **Never Asked:** empty-substring semantics, non-letter characters, whether output is length or the substring
**Self-Verified:** No — declared done without tracing, and commented out two of the three provided test cases including the one that exposes the bug. Never claimed an output.

## Phase Timings
| Phase | Budget | Actual | Hit? |
|---|---|---|---|
| Clarify | 4 min | 1 min (10:29:56) | Yes |
| Approach + dry run | 15 min | 33 min (11:01:51, both hints spent) | No |
| Code complete | 35 min | 63 min (11:31:51) | No |
| Test + complexity | 42 min | never reached | No |

---

## Problem Statement

Given a string `s`, return the length of the longest substring of `s` in which each of the five vowels — `a`, `e`, `i`, `o`, `u` — appears an even number of times. (Zero counts as even. Consonants are unconstrained.)

**Example 1**
```
Input:  s = "eleetminicoworoep"
Output: 13
Explanation: "leetminicowor" contains 2 e's, 2 o's, 2 i's and 0 a/u.
```

**Example 2**
```
Input:  s = "leetcodeisgreat"
Output: 5
Explanation: "leetc" contains 2 e's.
```

**Example 3**
```
Input:  s = "bcbcbc"
Output: 6
Explanation: the whole string has 0 of every vowel.
```

Constraints withheld; supplied on request: `1 <= s.length <= 5 * 10^4`, lowercase English letters only.

---

## Conversation Log

**Interviewer:** [10:28:45] Medium-Hard, 45 min. Presents problem with three examples, no constraints. "Any clarifying questions?"

**Aayush:** [10:29:56] what are the constraints ?

**Interviewer:** `1 <= s.length <= 5 * 10^4`, lowercase English letters only. Anything else, or do you want to talk through your approach?

**Aayush:** [10:33:38] brute force is O(n^3) for fixing [i:j] in i and another pass to scan the number of vowels in [i:j]. Can be reduced to O(n^2) by computing the vowel frequency map on the go.

**Interviewer:** O(n^2) at n = 5*10^4 — does that fit the budget you were given?

**Aayush:** [10:34:02] no the algorithm needs to be O(n) or nlogn at most

**Interviewer:** Right. So what's your approach?

**Aayush:** [10:49:09] we can use sliding pointer approach to reduce complexity to O(n)

**Interviewer:** We're 20 minutes in — approach checkpoint has passed. On sliding window: when your right pointer moves and the window becomes invalid, what tells you to move the left pointer, and by how much?

**Aayush:** can you give a hint

**Interviewer:** Before that — try answering the question I just asked. Take `"aeXea"`: right pointer reaches the end and the window has an odd `a` count. Where does left go?

**Aayush:** [10:52:53] there wont be odd count for a

**Interviewer:** Right, that one's all even — my example was bad. Try `"aeea"`. What's the leftmost valid left for right = 2, and for right = 3? [counterexample, not a hint]

**Aayush:** right = 2 left = 1 , right = 3 left=0

**Interviewer:** So what does that do to the sliding-window approach?

**Aayush:** [10:56:06] not understanding the core hint

**Interviewer:** [HINT 1 — approach checkpoint blown by >50%] Left moved backward — 1 then 0. A sliding window requires left to only advance, so that approach is dead here. Directional hint: whether `s[l..r]` is valid depends only on a small summary of prefix `[0..l-1]` and prefix `[0..r]`. Figure out what that summary is, and what relation between the two summaries makes the window valid.

**Aayush:** [10:59:14] we can store prefix vowel count for each of 5 the vowels. For a [l:r] to be valid prefix count of [0:r] - prefix count of [l-1:0] should be odd for all the 5 vowels.

**Interviewer:** Even or odd — which did you mean? And how does that give you O(n)?

**Aayush:** [11:01:51] even .

**Interviewer:** [HINT 2] "difference even for every vowel" means the two prefix parity-vectors are equal. There are only 32 such vectors. So as you scan, for each position you have a 5-bit parity state — and you want the earliest index where that same state occurred before. Write it up.

**Aayush:** [11:13:08] give me int main boilerplate code

**Interviewer:** [provides `class Solution` skeleton + `main` with the three statement examples] You're at 44 minutes — put down the best thing you have.

**Aayush:** [11:31:51] submits solution (below), with examples 1 and 3 commented out in `main`.

**Interviewer:** [11:31:51] Time's up — 63 minutes elapsed. Going to feedback.

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

class Solution {
public:
    int findTheLongestSubstring(string s) {
        int n = s.size();
        vector<vector<int>> prefixCharCnt(26,vector<int>(s.size(),0));

        prefixCharCnt[s[0]-'a'][0] = 1;
        for(int i=1;i<s.size();i++)
        {
            prefixCharCnt[s[i]-'a'][i] = prefixCharCnt[s[i]-'a'][i-1] + 1;
        }
        int ans = 0;
        map<string,int> mp;
        string initialState = "0|0|0|0|0";
        mp[initialState] = -1;
        for(int i=0;i<s.size();i++)
        {
            string prefixACntParity = to_string(prefixCharCnt['a'-'a'][i]%2);
            string prefixECntParity = to_string(prefixCharCnt['e'-'a'][i]%2);
            string prefixICntParity = to_string(prefixCharCnt['i'-'a'][i]%2);
            string prefixOCntParity = to_string(prefixCharCnt['o'-'a'][i]%2);
            string prefixUCntParity = to_string(prefixCharCnt['u'-'a'][i]%2);

            string parityState = prefixACntParity + "|" + prefixECntParity + "|" + prefixICntParity + "|" + prefixOCntParity + "|" + prefixUCntParity;
            if(mp.find(parityState) != mp.end())
            {
                ans = max(ans, i-mp[parityState]);
                continue;
            }
            mp[parityState] = i;
        }
        return ans;
    }
};
```

**Bug (never caught):** the prefix loop writes only the row of the character at index `i`; all other rows remain `0` at column `i`. So `prefixCharCnt[c][i]` is the length of the run of `c` ending at `i`, and `0` wherever `s[i] != c`. The parity state is therefore `0|0|0|0|0` at essentially every consonant, matching the seeded initial state, so `ans` grows to `i+1`.

Traced output on `"leetcodeisgreat"`: **15** (expected **5**).

**Optimal Solution:**
```cpp
int findTheLongestSubstring(string s) {
    vector<int> first(32, -2);   // -2 = unseen
    first[0] = -1;               // empty prefix has state 0 at index -1
    int state = 0, ans = 0;
    for (int i = 0; i < (int)s.size(); i++) {
        switch (s[i]) {
            case 'a': state ^= 1;  break;
            case 'e': state ^= 2;  break;
            case 'i': state ^= 4;  break;
            case 'o': state ^= 8;  break;
            case 'u': state ^= 16; break;
        }
        if (first[state] != -2) ans = max(ans, i - first[state]);
        else                    first[state] = i;
    }
    return ans;
}
```

**Time Complexity:** never stated by him (his code is O(n log n) with heavy string-construction constants; optimal is O(n)) · **Space Complexity:** never stated (his is O(26n); optimal is O(1) — 32 ints)

Note: he did get the subtlest correct detail right — storing the **first** occurrence and never overwriting (`continue` before the insert), plus seeding the empty-prefix state at `-1`.

---

## Feedback Given

### Round Conditions

**Hints used: 2/2 → hard ceiling of 2/5.**
- Hint 1 (27 min): validity of `[l..r]` is a relation between two prefix summaries.
- Hint 2 (33 min): the summaries are 5-bit parity vectors, only 32 of them, and you want the earliest prior occurrence of the same state. That was the algorithm.

**Constraints asked:** one generic "what are the constraints" (got `n <= 5*10^4`, lowercase only).
**Never asked:** whether the empty substring counts as an answer (it's why `0` is the floor and why the map is seeded with `-1` — you got that seed right by instinct, not by asking); whether `s` could contain non-letters; whether the answer wants length or the substring itself.

**Self-verified: No.** You submitted without tracing, and you commented out two of the three provided test cases — including the one from Example 1 that would have exposed the bug.

**Your code prints `15` on `"leetcodeisgreat"`. Expected `5`.**

The bug is in the prefix table: only the row for the character *at* position `i` is written; every other row stays `0` at column `i`. So `prefixCharCnt['e'][7]` isn't "e's seen so far" — it's "length of the run of `e` ending at 7", and it's `0` at any index where `s[i] != 'e'`. Your parity state is therefore `0|0|0|0|0` at nearly every consonant, which matches the seeded initial state, and `ans` grows to `i+1` — the whole prefix. Hence 15. A carried prefix needs every row copied forward, which is why nobody builds this as a 26xn table — you carry a single rolling state instead.

### Rubric

**Problem understanding & clarification — 2/5.** One generic constraint ask, no semantics. You did correctly derive the O(n) budget from `5*10^4` once you had it, which is progress over prior rounds. But "what are the constraints" is a checkbox, not a question — you didn't ask anything that would have changed your design.

**Approach & thought process — 1/5.** Brute force -> O(n^2) with rolling counts was fast and correct. Then you reached for sliding window, which is the reflex answer to "longest substring", and stayed there for 15 minutes. When I handed you `"aeea"` you correctly computed left = 1 for r=2 and left = 0 for r=3 — you had the refutation in your hand and didn't see that it killed your approach. Both hints then went to the core insight.

**Code quality & correctness — 1/5.** Broken on the very first example. Beyond the bug: `map<string,int>` with a 9-character key built by five `to_string` calls per index — ~10 allocations and a `log n` string-comparison lookup per character, when the state is five bits. O(26n) memory for five rows you needed.

**Complexity analysis — 1/5.** Never stated. I asked "how does that give you O(n)" at 30 minutes and you answered a different question. You also never noticed that what you wrote isn't O(n) — it's O(n log n) with a heavy constant.

**Communication — 2/5.** Clear when engaged, but 15 minutes of silence between "sliding window" and my prod, and "not understanding the core hint" instead of stating what you *had* understood and where it broke.

**Time management — 1/5.** Clarify by 4 min: hit (1 min). Approach by 15: missed — landed at 33, only with both hints. Code by 35: missed — submitted at 63. Test + complexity by 42: never reached.

### Performance Rating: 1/5

Ceiling analysis: heading for 2 on the code alone (bug never caught -> max 2), and the 2-hint ceiling also caps at 2. Lands at **1** because hint 2 *was* the algorithm — the core insight was given, not found.

---

### Algorithmic Thought-Process Debrief

**1. The derivation chain**

- **Step 0 — brute force.** For each `(l, r)`, count five vowels. O(n^3) -> O(n^2) with rolling counts. Done correctly and fast.
- **Step 1 — name the redundant work.** Recounting from `l` for every `r`. Standard fix: prefix sums. `count(v, l, r) = P[v][r] - P[v][l-1]`. Any window is now O(5). Still O(n^2) because you're enumerating pairs.
- **Step 2 — the condition is not a magnitude, it's a parity.** This is the step that changes the problem class. `P[v][r] - P[v][l-1] = 0 (mod 2)` <=> `P[v][r] = P[v][l-1] (mod 2)`. Subtraction becomes **equality**. The moment a range condition collapses into "two prefix summaries are equal," you are no longer searching pairs — you are looking up a key.
- **Step 3 — what is the minimal state?** Not the counts, only their parities: five bits. The entire prefix compresses to one of 32 values. That sentence ends the problem.
- **Step 4 — name the per-step operation, match the structure.** "Have I seen this exact state before, and where first?" -> hashmap; with a key space of 32, an `int mask[32]`.
- **Step 5 — earliest, not latest.** You want the longest window ending at `r`, so store the **first** index each state appeared and never overwrite. He got this right. Seed `mask[0] = -1` for the length-zero prefix.

**2. The signal he missed**

`"aeea"`, right = 2 -> left = 1; right = 3 -> left = 0. **Left moved backward.** That is a proof, not a hint: a two-pointer window is defined by left being monotone non-decreasing, so any problem where the optimal left regresses cannot be a sliding window. He computed the regression and then asked for a hint anyway.

Second missed signal, earlier and cheaper: **"even number of times" is a parity condition, not a magnitude condition.** Windows require a predicate monotone as the window grows ("sum >= k", "<= k distinct"). Parity is the opposite — it flips with every vowel. The word "even" should have disqualified the window before he proposed it.

**3. The generalization**

Tell: a condition on a subarray/substring expressible as `f(prefix_r) == f(prefix_{l-1})` for a small summary `f`. Move: scan once, maintain `f`, hashmap from `f` to earliest index (for longest) or to a count (for "how many").

| Condition | `f(prefix)` |
|---|---|
| Subarray sums to `k` | running sum, look up `sum - k` |
| Subarray sum divisible by `k` | `sum mod k` |
| Every vowel appears an even number of times | 5-bit parity mask (this problem) |
| Equal number of 0s and 1s | `count1 - count0` |
| Equal counts of every char in a fixed small alphabet | vector of differences from char 0 |

Counter-tell: **parity, XOR, and "divisible by" conditions are never sliding windows** — they're prefix-state-equality problems. Monotone conditions (sum with positives, distinct-count, max-min) are windows.

**4. Drill**

Three back-to-back in one sitting; before coding each, write one line: *"the summary f is ___, and I look up ___."*
1. Contiguous Array (LC 525) — longest subarray with equal 0s and 1s.
2. Subarray Sums Divisible by K (LC 974).
3. This problem, from scratch, bitmask, no prefix table.

Plus a standing 5-minute drill: before proposing sliding window, state the predicate and answer "is it monotone as the window grows?" out loud. He has now proposed a non-monotone window in two consecutive rounds.
