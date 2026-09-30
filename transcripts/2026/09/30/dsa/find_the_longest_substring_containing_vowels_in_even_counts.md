# DSA Round Transcript
**Date:** 2026-09-30
**Start Time:** 15:03:36 · **End Time:** 16:07:07 · **Duration:** 64 min
**Problem:** Find the Longest Substring Containing Vowels in Even Counts (LC 1371)
**Topic:** Prefix parity bitmask + hashmap of first occurrence
**Difficulty:** Medium-Hard
**Performance Rating:** 3/5  <!-- machine-read on future rounds; ≤2 = eligible for re-ask, ≥3 retired -->
**Hints Used:** 1/2
**Constraints Asked:** "what are the constraints?" (got length ≤ 5×10^5, lowercase only) · **Never Asked:** anything specific about semantics (uppercase vowels, empty string) — the generic question happened to cover it
**Self-Verified:** No. Skipped the requested dry-run of the final code (the code happens to be correct). On the first counterexample ("aea") he first asserted "returns 0" without tracing; the real output was 1. He self-corrected one message later.
**Would it have fit a real 45-min round?** No. At 45 min the approach had only just been settled, so he would have been cut off before writing any code.

## Phase Timings (untimed round — reference is a yardstick, not a gate)
| Phase | Reference | Actual | Delta | On pace? |
|---|---|---|---|---|
| Clarify | 4 min | +1 min (15:04:24) | −3 | On pace |
| Approach + dry run | 15 min | +45 min (15:48:46, bitmask settled) | +30 | Over by 30 min |
| Code complete | 35 min | +60 min (16:03:32) | +25 | Over by 25 min |
| Test + complexity | 42 min | +64 min (16:07:07) | +22 | Over by 22 min |
| **Total** | 45 min | 64 min | +19 | Over |

---

## Problem Statement
Given a string `s`, return the length of the longest substring in which each vowel ('a','e','i','o','u') appears an even number of times (0 counts as even).

Example 1: `s = "eleetminicoworoep"` → `13` ("leetminicowor")
Example 2: `s = "bcbcbc"` → `6`

---

## Conversation Log
**Interviewer (15:03:36):** Presented the problem as Medium-Hard, with the reference timeline marked as measured but not enforced. Asked for clarifying questions.
**Aayush:** what are the constraints ?
**Interviewer (+1m):** 1 ≤ s.length ≤ 5×10^5; lowercase English letters only. What's your approach?
**Aayush:** Constraints suggest O(n) or O(n log n). Brute force: fix i, j and check window → O(n³).
**Interviewer (+2m):** Agreed, O(n³) doesn't fit. Where do you take it?
**Aayush:** Reduce to O(n²) with precalculated prefix counts per vowel.
**Interviewer (+12m):** You said the budget is O(n)/O(n log n). Does O(n²) fit? What's next?
**Aayush:** For [i:j], prefixCnt[j][v]%2 == prefixCnt[i-1][v]%2 for every vowel. For a right index, the left boundary is the earliest left where all 5 parities match.
**Interviewer (+17m):** How do you find that earliest left, and at what cost per lookup?
**Aayush:** map<vowel, map<parity, earliestIndex>>. Get 5 candidate lefts (one per vowel) and take the maximum.
**Interviewer (+26m):** Dry-run on "aea". What does it return?
**Aayush:** Need to handle "no earliest index" for a vowel. With that it returns 0.
**Interviewer (+36m):** Okay, you say 0. Code it.
**Aayush:** No, it returns 1 but the answer should be 0. Taking the max among the 5 per-vowel indices is the wrong choice.
**Interviewer (+38m):** Okay. How do you fix it?
**Aayush:** Combine the 5 prefix sums into one integer and compare that.
**Interviewer (+40m):** Be specific: how are they combined, what's the range, and what does equality mean?
**Aayush:** (pc[i][a]+pc[i][e]+…+pc[i][u])%2 == (pc[j][a]+…)%2.
**Interviewer (+41m):** Dry-run on "ae". What does it return?
**Aayush:** 2, which is wrong.
**Interviewer (+43m):** Right. Where are you leaning now?
**Aayush:** can you give a hint
**Interviewer (+43m) [HINT 1]:** The sum throws away *which* vowel is odd; you need one value that keeps all five parities separate. What would it look like, and how many values can it take?
**Aayush:** 5-bit bitmask, 32 values; map mask → earliest index.
**Interviewer (+45m):** Code it.
**Aayush:** Requested C++ boilerplate. **Interviewer:** Provided (only the two statement examples in main).
**Aayush (+60m):** Submitted code. TC O(n), SC O(1) since the map has ≤32 keys.
**Interviewer (+60m):** (1) Dry-run on "aea" with the mask at each index. (2) Walk every allocation. Does O(1) space hold?
**Aayush:** (answered only 2) Missed the O(n) prefix-count arrays.
**Interviewer (+61m):** So O(n)/O(n); dry-run skipped, code taken as submitted. Can it be optimized further?
**Aayush:** Drop the prefix arrays; keep 5 running parities toggled 0/1 → O(1) space.
**Aayush (+64m):** Asked which LeetCode problem this is. **Interviewer:** LC 1371.

---

## Solution
**Aayush's Final Solution:**
```cpp
class Solution {
public:
    int findTheLongestSubstring(string s) {
        int n = s.size();
        unordered_map<int,int> mp;
        map<char,vector<int>> cnt;
        cnt['a'] = vector<int>(n,0);
        cnt['e'] = vector<int>(n,0);
        cnt['i'] = vector<int>(n,0);
        cnt['o'] = vector<int>(n,0);
        cnt['u'] = vector<int>(n,0);
        vector<char> vowels({'a','e','i','o','u'});
        int ans = 0;
        mp[0] = -1;
        for(int i=0;i<n;i++)
        {
            for(char ch:vowels)
                if(i>0) cnt[ch][i] = cnt[ch][i-1];
            if(find(vowels.begin(),vowels.end(),s[i]) != vowels.end())
                cnt[s[i]][i]++;
            int mask = 0;
            for(int bit=0;bit<5;bit++)
                mask |= ((cnt[vowels[bit]][i]%2)<<bit);
            if(mp.find(mask) == mp.end()) { mp[mask] = i; continue; }
            ans = max(ans, i-mp[mask]);
        }
        return ans;
    }
};
```
Verified by the interviewer after the round: **correct**. "eleetminicoworoep" gives 13, "bcbcbc" gives 6, "aea" gives 0, and "ae" gives 0.

**Optimal Solution:**
```cpp
class Solution {
public:
    int findTheLongestSubstring(string s) {
        int first[32];
        fill(first, first + 32, INT_MIN);
        first[0] = -1;                       // empty prefix has mask 0
        int mask = 0, ans = 0;
        for (int i = 0; i < (int)s.size(); i++) {
            switch (s[i]) {
                case 'a': mask ^= 1;  break;
                case 'e': mask ^= 2;  break;
                case 'i': mask ^= 4;  break;
                case 'o': mask ^= 8;  break;
                case 'u': mask ^= 16; break;
            }
            if (first[mask] == INT_MIN) first[mask] = i;
            else ans = max(ans, i - first[mask]);
        }
        return ans;
    }
};
```
**Time Complexity:** his answer O(n) ✔ · **Space Complexity:** his answer O(1) ✘ at first (missed the 5×n arrays), corrected to O(n) when asked; then he reached O(1) himself with running parities.

---

## Feedback Given

### Round conditions
- **Hints used: 1/2.** This caps the rating at 3. The hint was "the sum throws away which vowel is odd; keep all five separate". It pointed straight at the bitmask.
- **Constraints:** he asked "what are the constraints?" at +1m and immediately turned n = 5×10^5 into an O(n)/O(n log n) budget. That's a clear improvement. He never asked about semantics directly; the generic question happened to cover the character set.
- **Self-verification:** none. On "aea" he first *asserted* the output was 0 (real: 1), then retracted one message later. He skipped the requested dry-run of the final code. The code happens to be correct, but he didn't show it.

### Rubric
- **Problem understanding & clarification: 4/5.** Asked early and turned the answer into a budget. Nothing asked about semantics beyond that.
- **Approach & thought process: 3/5.** He reached the key insight, parity equality on prefixes, **unaided**. That's the hard part. Then he combined the five conditions wrongly twice: first max-of-per-vowel-lefts, then parity of the sum. He asserted each without proof, and a counterexample broke each one. The final encoding (the bitmask) came from the hint.
- **Code quality & correctness: 3/5.** Correct, but heavyweight: 5 prefix arrays of size n, a `map<char,vector>`, and `find` on every character, where one running int does the job. No self-test.
- **Complexity analysis: 3/5.** Time was right. He claimed O(1) space while allocating 5n ints, fixed it when asked, then found the O(1) optimization himself.
- **Communication: 2/5.** Three gaps of 9–10 min (+2→+12, +17→+26, +26→+36), each ending in a one-line idea. He asserted "returns 0" instead of tracing. He asked for a hint when asked where he was leaning.
- **Time management: 1/5.** See the pace report.

### Pace report
| Phase | Ref | Actual | |
|---|---|---|---|
| Clarify | 4 | 1 | on pace |
| Approach | 15 | 45 | over by 30 |
| Code | 35 | 60 | over by 25 |
| Test + complexity | 42 | 64 | over by 22 |

**Would it have fit a real 45-min round? No.** A real interviewer would have stopped him during the approach phase: at 45 min he had only just named the bitmask and had written no code. The biggest time sink was **+2m → +36m**. About 34 minutes went on three sequential ideas (prefix counts, parity, max-of-5). Each took about 10 minutes and none was tested before he spoke it. The 10-minute stretch that ended in "it returns 0" was pure loss.

**Performance Rating: 3/5.** This lands at 3, which is also the binding ceiling from one hint used. Without the hint, the unaided parity insight and correct code would still not have lifted it past 3 at 64 minutes.

### Algorithmic thought-process debrief
1. **Derivation chain**
   - Brute force: O(n³) check per window. The redundant work is recounting vowels, so use prefix counts to get O(n²). The redundant work is now trying every left.
   - Fix the right end. The condition "count in window is even" becomes "prefix parity at L equals prefix parity at R". Now the question is "have I seen this state before, and where first?"
   - The state is *five* parities that must **all** match at the **same** L, so the key must be the whole tuple, not per-vowel pieces.
   - Five bits give a 5-bit mask (32 states). Toggle with XOR. Store "first index seen" in a hashmap or a 32-slot array. That's O(n) time and O(1) space.
2. **Signal he missed:** "for **all** 5 vowels, the **same** left". A conjunction over the same index means the lookup key must be the conjunction itself. The moment he wrote "for all 5 vowels" at +17m, the tuple *was* the key. Splitting it into per-vowel maps, then collapsing it with a sum, both destroy that. Before merging conditions, ask: does this merge keep equality exact, both ways?
3. **Generalization:** "longest or count of subarrays where some *per-element property* balances out" means prefix state plus first-occurrence (or count) hashmap. When the state is several small independent flags (parities, a set of ≤ ~20 characters), the tell is **"each of k things is even/odd/present"**, which means a k-bit mask with XOR. Relatives: LC 1915 (Wonderful Substrings), LC 1542 (Longest Awesome Substring), LC 525 (Contiguous Array), LC 560.
4. **Drill:** LC 1915 (Number of Wonderful Substrings), ≤ 20 min. Before coding, write the state key and prove in one sentence that key equality ⇔ window validity, in both directions. Then run it by hand on one input you invent yourself.
