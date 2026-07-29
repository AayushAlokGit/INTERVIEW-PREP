# DSA Round Transcript
**Date:** 2026-06-19
**Start Time:** 10:16
**End Time:** 11:01
**Duration:** 45 minutes
**Problem:** Partition Labels
**Topic:** Greedy / Hashing (last-occurrence sweep)
**Difficulty:** Medium

---

## Problem Statement
You're given a string `s` of lowercase English letters. Partition the string into as many parts as possible so that each letter appears in at most one part (no letter spans two parts). Return a list of the part sizes, in order.

**Example 1:**
```
s = "ababcbacadefegdehijhklij"
Output: [9, 7, 8]   -> "ababcbaca", "defegde", "hijhklij"
```

**Example 2:**
```
s = "eccbbbbdec"
Output: [10]
```

**Constraints:**
- `1 <= s.length <= 500`
- `s` consists of lowercase English letters only (<= 26 distinct chars).

---

## Conversation Log

**Interviewer:** Presented the problem; asked for clarifying questions.

**Aayush:** What are the constraints?

**Interviewer:** Provided constraints (len <= 500, lowercase only, <= 26 distinct).

**Aayush:** Insight — first and last occurrence of each char is crucial; each char must be in at most one partition, so the minimal valid partition must include both its first and last occurrence. Build a [first, last] interval per distinct char, merge overlapping intervals, and the number of merged intervals is the answer.

**Interviewer:** Confirmed correct (each part size = last - first + 1). Asked: (1) do you actually need an explicit sort to merge, given the order first occurrences appear? (2) Can you produce the answer in a single left-to-right pass without materializing/merging all 26 intervals — what's the minimal state?

**Aayush:** (1) First occurrences are already in sorted order. (2) Proposed: iterate L->R maintaining the current partition and a map of char -> partition number of its first occurrence; partNum starts at 1. New char -> partNum++, mp[c]=partNum. Re-encountered char -> if current partNum > mp[c], set partNum = mp[c] and merge all chars with partition > mp[c] back into mp[c].

**Interviewer:** Flagged this as the generic/fiddly path (retroactive back-merging, relabeling the map). Hinted: flip from first-occurrence to LAST-occurrence — precompute each char's last index in one pass, sweep, and track "the furthest index you're still obligated to reach." Asked for the minimal state and the exact close condition.

**Aayush:** At each index i, partEnd = max(lastOccurrence[s[i]], partEnd); if partEnd == i then a new partition must start at i.

**Interviewer:** Confirmed. Asked him to write full code including how sizes are computed/collected.

**Aayush:** Wrote the C++ solution (reverse pass to fill last-occurrence map, forward sweep with partStart/partEnd, push parts, size = end - start + 1).

**Interviewer:** Asked: (1) precise complexity given std::map usage and what to swap; (2) is resetting partEnd to i+1 necessary or does max() handle it — why is it safe; (3) one edge case + its output.

**Aayush:** (1) O(n log n) with map, O(n) with unordered_map. (2) max() handles it since last[s[i+1]] >= i+1 and partEnd = i, so max auto-updates. (Skipped #3.)

**Interviewer:** Pointed out #3 was dropped (recurring habit). Corrected #1: map holds <= 26 keys, so log factor is over a constant -> actually O(n); cleanest is int last[26] for unambiguous O(n)/O(1). Re-asked for the edge case.

**Aayush:** All-distinct -> "abc" -> output 1,1,1.

**Interviewer:** Correct, but noted that's the shallowest case; a stressier one is "abac" -> [3,1], which actually exercises the max logic. Wrapped up.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s="aaaaabccdd";
    int n = s.size();
    vector<pair<int,int>> parts;
    map<char,int> last;
    for(int i=n-1;i>=0;i--)
    {
        if(last.find(s[i]) == last.end())
        {
            last[s[i]] = i;
        }
    }
    int partEnd = 0;
    int partStart = 0;
    for(int i=0;i<n;i++)
    {
        partEnd = max(last[s[i]], partEnd);
        if(i==partEnd)
        {
            parts.push_back({partStart, partEnd});
            partStart = partEnd+1;
            partEnd = partEnd+1;
        }
    }
    for(auto p:parts)
    {
        cout<<p.first<<"-"<<p.second<<" size -> "<<p.second-p.first + 1<<endl;
    }
    return 0;
}
```

**Optimal Solution (idiomatic, array for O(n)/O(1)):**
```cpp
vector<int> partitionLabels(string s) {
    int last[26];
    for (int i = 0; i < (int)s.size(); ++i) last[s[i]-'a'] = i;
    vector<int> res;
    int start = 0, end = 0;
    for (int i = 0; i < (int)s.size(); ++i) {
        end = max(end, last[s[i]-'a']);
        if (i == end) {
            res.push_back(i - start + 1);
            start = i + 1;
        }
    }
    return res;
}
```

**Time Complexity:** O(n) (the std::map log factor is over <= 26 keys = constant).
**Space Complexity:** O(1) extra (26-entry last-occurrence table) + O(#parts) output.

---

## Feedback Given

**Time Taken: 45 minutes**

### Problem Understanding & Clarification — 7/10
Asked for constraints unprompted; immediately latched onto the [first,last]-per-char structural fact. Fast, accurate read.

### Approach & Thought Process — 7/10
First instinct (build all intervals + merge) correct but generic. When pushed for the lean pass, doubled down on a MORE convoluted version (partition numbers + retroactive back-merging) — the structure-exploiting weakness recurring. One hint (flip to last-occurrence) and he snapped to the optimal partEnd = max(...) sweep with a precise close condition. Insight reachable; default still needs retraining. Lesson: when the first approach feels fiddly, treat the friction as a signal to find the cleaner invariant.

### Code Quality & Correctness — 9/10
Clean, correct first write. Reverse pass for last-occurrence, single forward sweep, correct size calc and reset. No bugs. Minor: std::map over int[26].

### Complexity Analysis — 6.5/10
Initially said O(n log n) for the map version — recurring mis-attribution of the log factor. Map holds <= 26 keys so log is over a constant -> O(n). Did note unordered_map/array gives clean O(n).

### Communication — 6.5/10
Positive: crisp, correct justification of why the partEnd reset is safe (last[s[i+1]] >= i+1). Negative: dropped sub-question #3 (edge case) twice in one session — same habit as last round. And the edge case eventually given ("abc", all-distinct) was the shallowest, not stressing the max logic.

### What to drill
1. Friction = signal: when the first approach gets fiddly (back-merging/relabeling), stop and hunt the cleaner invariant.
2. Answer every sub-question — dropped one twice this session.
3. Edge cases should stress the logic (overlap/nesting/all-same), not restate the happy path.
4. Don't bolt log n onto alphabet-bounded structures — that's a constant.

**Overall: ~7.2/10** — good outcome (optimal solution, clean code), but re-exposed two tracked habits: generic-over-structure default, and dropping sub-questions.
