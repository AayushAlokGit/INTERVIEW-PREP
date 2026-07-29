# DSA Round Transcript
**Date:** 2026-05-04
**Start Time:** 12:48
**End Time:** 13:14
**Duration:** 26 minutes
**Problem:** Longest Substring with At Most K Distinct Characters
**Topic:** Sliding Window
**Difficulty:** Medium

---

## Problem Statement
Given a string `s` and an integer `k`, return the length of the longest substring of `s` that contains at most `k` distinct characters.

**Example 1:**
```
Input: s = "eceba", k = 2
Output: 3
Explanation: "ece" contains 2 distinct characters.
```

**Example 2:**
```
Input: s = "aa", k = 1
Output: 2
```

**Example 3:**
```
Input: s = "abcadcacacaca", k = 3
Output: 11
```

**Constraints:**
- 1 <= s.length <= 5*10^4
- 0 <= k <= 50
- s consists of English letters.

---

## Conversation Log

**Interviewer:** Presented problem, asked for clarifying questions.

**Aayush:** What would be the case when k = 0 — is the answer always 0?

**Interviewer:** Yes, exactly.

**Aayush:** Brute force: scan each substring (O(N^2)), check distinct chars per substring (O(N)). Total O(N^3) time, O(N) space.

**Interviewer:** Correct. Can you do better?

**Aayush:** Sliding window with l, r pointers. R increases while distinct chars <= k. L increases when distinct > k. Hashmap of char counts; map size = distinct count.

**Interviewer:** Confirmed approach. Asked clarifying questions about expand condition, what happens to map on shrink, where to update answer.

**Aayush:** R increases when distinct <= k, L increases when distinct > k, decrement freq of s[l] before incrementing. Update answer when r increases.

**Interviewer:** Noted: must erase key when count hits 0. Asked him to code.

**Aayush (Attempt 1):**
```cpp
while(r<n) {
    if(mp.size() <= k) {
        ans = max(ans, r-l+1);
        r++;
        mp[s[r]]++;
    }
    while(l<n && mp.size() > k) {
        mp[s[l]]--;
        if(mp[s[l]] == 0) mp.erase(s[l]);
        l++;
    }
}
```

**Interviewer:** Pointed out OOB on `s[r]` after `r++`, init issues, traced through.

**Aayush (Attempt 2):** Restructured but still had off-by-one — added `r<n` check inside if, but `ans = r-l+1` after `r++` over-counts.

**Interviewer:** Traced "eceba" k=2: after first iter, ans=2 but window is just "e" (length 1). Bug: after r++, window length = r-l, not r-l+1.

**Aayush (Attempt 3 — Correct):**
```cpp
while (r < n) {
    mp[s[r]]++;
    r++;
    while (mp.size() > k) {
        mp[s[l]]--;
        if (mp[s[l]] == 0) mp.erase(s[l]);
        l++;
    }
    ans = max(ans, r - l);
}
```

**Interviewer:** Traced and confirmed correct. Asked for complexity.

**Aayush:** O(N) time (each char visited at most twice), O(N) space (hashmap stores chars).

**Interviewer:** Time correct. Probed space — algorithm shrinks when size > k, so map holds at most k+1.

**Aayush:** Space is O(1) since bounded by English alphabet.

**Interviewer:** Valid, but tighter algorithm-derived bound is O(k). Prefer algorithm-derived bounds.

**Interviewer:** Any further optimization?

**Aayush:** I think it is tight.

**Interviewer:** Correct — O(N) is optimal. Noted unordered_map vs map (O(1) avg vs O(log k)).

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s;
    int n = s.size();
    int k;

    map<char,int> mp;
    int l=0, r=0, ans=0;

    if(k==0) return ans;

    while (r < n) {
        mp[s[r]]++;
        r++;
        while (mp.size() > k) {
            mp[s[l]]--;
            if (mp[s[l]] == 0) mp.erase(s[l]);
            l++;
        }
        ans = max(ans, r - l);
    }
    cout << ans;
    return 0;
}
```

**Optimal Solution:** Same as above (use `unordered_map` for O(1) avg ops).

**Time Complexity:** O(N) — each character enters/leaves window at most once.
**Space Complexity:** O(k) — algorithm caps map at k+1 entries.

---

## Feedback Given

| Category | Score | Notes |
|---|---|---|
| Problem Understanding & Clarification | 4/5 | Asked k=0 edge. Missed: charset, k > alphabet. |
| Approach & Thought Process | 4.5/5 | Brute → optimal cleanly. Correct intuition on hashmap. |
| Code Quality & Correctness | 3/5 | Two iterations with off-by-one and OOB bugs before correct. Uninitialized `n = s.size()` before reading `s`. |
| Complexity Analysis | 3.5/5 | Time correct (amortized). Space O(1) via alphabet — correct but should derive O(k) from algorithm. |
| Communication | 4/5 | Clear narration. Responsive to feedback. Should trace before submitting. |

**Overall: ~3.8/5**

### Key takeaways
1. Standard sliding window template: expand r, shrink while invalid, update ans at end.
2. After `r++`, window length = `r - l`, not `r - l + 1`.
3. State space in terms of algorithm invariants (O(k)) over external constants (O(1) via alphabet).

**Time Taken: 26 minutes**
