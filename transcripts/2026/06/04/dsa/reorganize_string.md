# DSA Round Transcript
**Date:** 2026-06-04
**Start Time:** 9:36
**End Time:** 10:15
**Duration:** 39 minutes
**Problem:** Reorganize String
**Topic:** Greedy + Heap (Priority Queue)
**Difficulty:** Medium

---

## Problem Statement
Given a string `s`, rearrange the characters of `s` so that no two adjacent characters are the same. Return any valid rearrangement that satisfies this condition. If it is not possible, return an empty string `""`.

**Example 1:**
```
Input:  s = "aab"
Output: "aba"
```

**Example 2:**
```
Input:  s = "aaab"
Output: ""        (no valid rearrangement exists)
```

**Constraints:**
- `1 <= s.length <= 500`
- `s` consists of lowercase English letters only (`a`–`z`)

---

## Conversation Log

**Interviewer:** Presented the problem and asked for clarifying questions before approach.

**Aayush:** "what are the constraints?"

**Interviewer:** Gave constraints (length 1..500, lowercase letters only, alphabet bounded at 26).

**Aayush:** Proposed a backtracking approach — `solve(int i)` returns true/false depending on whether a rearrangement is possible. Keep a count of characters in the original string. Return false if no chars left to add but not all consumed. At index `i`, try every char that differs from the `i-1`th char and still has remaining count; if any leads to a valid arrangement return true, else false.

**Interviewer:** Confirmed the brute-force is correct, then asked: (1) worst-case complexity with n up to 500, and (2) whether a greedy choice exists instead of searching.

**Aayush:** "TC is O(26^n) exponential, n = size of string s. Greedy approach — for deciding current character we can choose the character different from last and having the maximum frequency of all available."

**Interviewer:** Asked (1) why "most frequent" is correct and (2) what data structure efficiently yields the max-frequency char different from last.

**Aayush:** "1. Picking the most frequent allows us to put the most occurring chars close together and alternating. 2. MaxHeap."

**Interviewer:** Asked how to avoid placing the same char as `last` while using the heap, and when to detect impossibility.

**Aayush:** "Start iteration from i=1 and maintain a last character for the previously placed char. While heap is not empty and heap top == last keep popping and putting in a temp array. If heap is empty then return empty string. Else assign the heap top as the ith character and update last. Finally if temp is non empty push back elements to the heap."

**Interviewer:** Noted the heap stores distinct chars, asked how many entries can equal `last` at once, then asked him to code it.

**Aayush:** Submitted the C++ solution (below). Stated "TC is O(nlogn) and SC is O(n)."

**Interviewer:** Pushed on complexity (heap bounded by 26 → is the log really log n?) and asked for a dry-run of the impossible case `"aaab"`.

**Aayush:** "max 26 elements since at max 26 lowercase english letters." Traced `"aaab"`: i=0 ans={a} heap={a,2}{b,1}; i=1 ans={a,b} heap={a,2}; i=2 ans={a,b,a} heap={a,1}; final iteration → empty string returned.

**Interviewer:** Confirmed trace correct, asked for the precise time complexity given the 26-element bound.

**Aayush:** "O(n)."

**Interviewer:** Confirmed, asked for end time, delivered feedback.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    string s="aaab";
    int n = s.size();
    string ans="";
    vector<int> cnt(26,0);

    for(char ch:s)cnt[ch-'a']++;

    priority_queue<pair<int,char>> maxH;

    for(int i=0;i<26;i++)
    {
        if(cnt[i] > 0) maxH.push({cnt[i], 'a' + i});
    }

    char last;
    int lastCnt;

    for(int i=0;i<n;i++)
    {
        if(i==0)
        {
            lastCnt = maxH.top().first;
            last = maxH.top().second;
            maxH.pop();
            ans.push_back(last);
            if(lastCnt > 1) maxH.push({lastCnt-1, last});
            continue;
        }

        vector<pair<int,char>> tmp;

        while(!maxH.empty() && last == maxH.top().second)
        {
            tmp.push_back(maxH.top());
            maxH.pop();
        }

        if(maxH.empty())
        {
            cout<<"ans is ";   // impossibility detected — should return ""
            return 0;
        }

        lastCnt = maxH.top().first;
        last = maxH.top().second;
        maxH.pop();
        ans.push_back(last);
        if(lastCnt > 1) maxH.push({lastCnt-1, last});

        if(tmp.size() > 0)
        {
            for(auto p:tmp)maxH.push(p);
        }
    }

    cout<<"ans is "<<ans;
    return 0;
}
```

**Optimal Solution (same approach — this is optimal):**
Greedy with a max-heap keyed on frequency. At each position place the highest-count available char that differs from the previous; hold the forbidden char aside, place the next-best, then push the held char back. Impossible iff at some step the only remaining char equals `last`. Heap is bounded by the 26-letter alphabet, so each operation is O(log 26) = O(1).

Notes / refinements:
- In a real function, the impossible branch should `return ""` (not print).
- The `tmp` vector can only ever hold **one** element (heap holds distinct chars, so at most one equals `last`) — a single held-aside variable suffices.

**Time Complexity:** O(n) — heap operations are O(log 26) = O(1); n iterations. (Initially stated O(n log n), corrected to O(n).)
**Space Complexity:** O(n) for the output string; O(1) auxiliary (cnt array + heap bounded by 26).

---

## Feedback Given

### What you did well
- Clarifying questions improved markedly — explicitly asked for constraints and value ranges before approaching.
- Strong approach progression: correct brute-force backtracking → identified exponential cost → independently arrived at greedy "most-frequent-available, different-from-last" and justified why it works → chose max-heap.
- Clean heap mechanics: pop-into-temp / place / push-back pattern correct; impossibility detection right.
- Correct dry-run when asked; self-corrected complexity once prodded.

### What to sharpen
1. **Complexity precision (recurring).** First stated O(n log n). Heap is bounded by the 26-letter alphabet, so the log factor is constant → true cost O(n). Reached it only after prompting. Make "how big can this structure actually get?" a reflex before stating Big-O.
2. **Trace before declaring done.** Announced TC/SC the moment typing finished without volunteering a trace; the impossibility path is exactly where a quick run pays off. In a real function, the impossible branch should cleanly `return ""`, not print.
3. **Minor:** the `while`-into-`tmp` can only ever hold one element (distinct chars in heap) — could simplify to a single variable.

### Scores (out of 5)
| Criterion | Score | Note |
|---|---|---|
| Problem understanding & clarification | 5 | Asked constraints + ranges upfront |
| Approach & thought process | 5 | Brute-force → greedy → heap, well justified |
| Code quality & correctness | 4.5 | Correct; tighten impossible-case return + temp logic |
| Complexity analysis | 3.5 | Overstated log factor; corrected only when probed |
| Communication | 4.5 | Clear, traced accurately on request |

**Overall: 4.5 / 5** — a strong, optimal solve. The one consistent leak is complexity precision: pause to bound your data structures before stating Big-O.

**Time Taken: 39 minutes**
