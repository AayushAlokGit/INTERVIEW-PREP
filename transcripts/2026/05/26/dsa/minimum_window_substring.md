# DSA Round Transcript
**Date:** 2026-05-26
**Start Time:** 9:32
**End Time:** 10:06
**Duration:** 34 minutes
**Problem:** Minimum Window Substring
**Topic:** Sliding Window / Hash Map
**Difficulty:** Hard

---

## Problem Statement
Given two strings `s` and `t`, return the **minimum window substring** of `s` such that every character in `t` (including duplicates) is included in the window. If there is no such substring, return the empty string `""`.

The answer is guaranteed to be unique.

**Example 1:**
```
Input:  s = "ADOBECODEBANC", t = "ABC"
Output: "BANC"
```

**Example 2:**
```
Input:  s = "a", t = "a"
Output: "a"
```

**Example 3:**
```
Input:  s = "a", t = "aa"
Output: ""
```

**Constraints:**
- 1 <= |s|, |t| <= 10^5
- English letters (52 chars max)
- Answer guaranteed unique if it exists

---

## Conversation Log

**Interviewer:** Presented the problem and asked for clarifying questions.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints (1 <= |s|, |t| <= 10^5, English letters, unique answer).

**Aayush:** Can t be longer than s?

**Interviewer:** Yes, in which case return "".

**Aayush:** Brute force — iterate over all windows of size >= |t| in s and check each via frequency maps. TC roughly O(|s| * |s| * |t|).

**Interviewer:** Confirmed brute force is ~O(|s|² · (|s|+|t|)). Asked for better.

**Aayush:** Sliding window with two pointers l, r. Expand r updating freq map. When window contains all chars of t (tracked via a `cnt` of how many distinct chars of t are satisfied), shrink l. Maintain min.

**Interviewer:** Summarized back and confirmed. Asked for TC/SC and to code.

**Aayush:** TC O(n), SC O(1).

**Interviewer:** Go ahead.

**Aayush:** [submitted first version of code — bug in shrink condition: decremented cnt only when freq[s[l]] == 0, which is wrong for duplicate chars in t]

**Interviewer:** Asked Aayush to trace example 3 (`s="a", t="aa"`) and to think about whether his code handles `t = "AABC"` correctly.

**Aayush:** [Submitted fixed code — changed shrink condition to `freq[s[l]] < tFreq[s[l]]`. Did not address example 3.]

**Interviewer:** Walked through example 3: start = -1, len = INT_MAX, substr(-1, INT_MAX) is undefined behavior. Asked how to handle.

**Aayush:** If start == -1 return empty string.

**Interviewer:** Asked for final TC/SC and optimization.

**Aayush:** TC O(n), SC O(k <= 52). Space optimization: array of size 52 instead of map.

**Interviewer:** Pointed out std::map is O(log k) per op → actual TC is O(n log k). unordered_map or array gives O(n).

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s = "ADOBECODEBANC";
    string t = "AABC";

    if (t.size() > s.size()) { cout << "ans is ''"; return 0; }

    map<char,int> tFreq;
    for (char ch : t) tFreq[ch]++;

    map<char,int> freq;
    int l = 0, r = 0;
    int len = INT_MAX;
    int start = -1;
    int cnt = 0;

    while (r < s.size()) {
        freq[s[r]]++;
        if (tFreq.find(s[r]) != tFreq.end()) {
            cnt += (freq[s[r]] == tFreq[s[r]]);
        }
        r++;
        while (cnt == (int)tFreq.size()) {
            int cur_len = r - l;
            if (cur_len < len) {
                len = cur_len;
                start = l;
            }
            freq[s[l]]--;
            if (tFreq.find(s[l]) != tFreq.end() && freq[s[l]] < tFreq[s[l]]) {
                cnt--;
            }
            l++;
        }
    }

    if (start == -1) { cout << "ans is ''"; return 0; }
    cout << "ans is " << s.substr(start, len);
    return 0;
}
```

**Optimal Solution (same pattern with array for O(1) ops):**
```cpp
string minWindow(string s, string t) {
    if (t.size() > s.size()) return "";
    int tFreq[128] = {0}, freq[128] = {0};
    int distinct = 0;
    for (char c : t) {
        if (tFreq[c] == 0) distinct++;
        tFreq[c]++;
    }
    int l = 0, cnt = 0, bestLen = INT_MAX, bestStart = -1;
    for (int r = 0; r < (int)s.size(); r++) {
        char c = s[r];
        freq[c]++;
        if (tFreq[c] > 0 && freq[c] == tFreq[c]) cnt++;
        while (cnt == distinct) {
            if (r - l + 1 < bestLen) { bestLen = r - l + 1; bestStart = l; }
            freq[s[l]]--;
            if (tFreq[s[l]] > 0 && freq[s[l]] < tFreq[s[l]]) cnt--;
            l++;
        }
    }
    return bestStart == -1 ? "" : s.substr(bestStart, bestLen);
}
```

**Time Complexity:** O(|s| + |t|) with array; O((|s|+|t|) log k) with std::map (as written)
**Space Complexity:** O(k) bounded by alphabet size (52/128) → O(1)

---

## Feedback Given

### Scoring (out of 5)

| Category | Score | Notes |
|---|---|---|
| Problem understanding & clarification | 4 / 5 | Asked about constraints and t > s upfront. Didn't ask about case sensitivity or impossible-case return. |
| Approach & thought process | 4.5 / 5 | Brute force first with complexity, then natural progression to sliding window with cnt-matching. Clean reasoning. |
| Code quality & correctness | 2.5 / 5 | Two bugs needing prompting: duplicate-char shrink condition (freq==0 vs freq<tFreq), and substr crash on no-window case. Didn't dry-run. |
| Complexity analysis | 3 / 5 | Said O(n) but std::map is O(log k) per op. Missed until prompted. Space analysis fine once stated. |
| Communication | 4 / 5 | Clear narration. Didn't volunteer edge cases unprompted. |

### Highlights
- Identified sliding-window with cnt-matches pattern naturally — the canonical optimal here.
- Quickly fixed duplicate-char shrink bug when nudged.
- Clean code structure, good naming.

### Areas to work on
- Dry-run before declaring done — would have caught both bugs in seconds.
- Container choice affects complexity — std::map vs unordered_map vs array. State it precisely.
- Volunteer the "no answer" guard for sliding-window-min problems.

**Time Taken: 34 minutes**
