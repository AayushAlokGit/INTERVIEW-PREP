# DSA Round Transcript
**Date:** 2026-05-05
**Start Time:** 12:39
**End Time:** 13:21
**Duration:** 42 minutes
**Problem:** Word Break
**Topic:** Dynamic Programming, Backtracking
**Difficulty:** Medium

---

## Problem Statement

Given a string `s` and a dictionary of strings `wordDict`, return `true` if `s` can be segmented into a space-separated sequence of one or more dictionary words. The same word may be reused multiple times.

**Example 1:**
```
Input:  s = "leetcode", wordDict = ["leet", "code"]
Output: true
```

**Example 2:**
```
Input:  s = "applepenapple", wordDict = ["apple", "pen"]
Output: true
```

**Example 3:**
```
Input:  s = "catsandog", wordDict = ["cats", "dog", "sand", "and", "cat"]
Output: false
```

**Constraints:**
- 1 <= s.length <= 300
- 1 <= wordDict.length <= 1000
- 1 <= wordDict[i].length <= 20
- s and wordDict[i] consist of only lowercase English letters
- All strings in wordDict are unique

---

## Conversation Log

**Interviewer:** Please note the current time and share it. Here is your problem: Word Break. [Problem presented as above.]

**Aayush:** 12:39

**Interviewer:** Do you have any clarifying questions?

**Aayush:** What are the constraints?

**Interviewer:** [Shared constraints listed above.]

**Aayush:** I am thinking of a recursive approach where we make sections of s. We keep 2 pointers last and i. At i we check if s[last:i] is present in wordDict or not. If it is present then we can add a break here, so last = i and i = i+1, else i can't move further we return.

**Interviewer:** What does "i can't move further, we return" mean exactly? What's your base case? What's the recursive structure when you find a valid word?

**Aayush:** Base case: i == n → empty string always true. At (last, i) if we can break at i to get valid s[last:i] then recurse for either (i, i+1) or (last, i+1). At (last, i) if we can't break then recurse to (last, i+1).

**Interviewer:** The "or" is confusing — are you trying both options (backtracking)? And when i == n but last != n, is the base case correct?

**Aayush:** 1. Trying both options. 2. Base case: s[last:n] not in wordDict → false, else true.

**Interviewer:** Good. Go ahead and code it up.

**Aayush:** [Submitted backtracking code — see Solution section below.]

**Interviewer:** Good effort. isPresent is O(n) linear scan — what data structure would make it O(1)? Also s is passed by value. And there's no memoization — that's the big issue. Can you trace through Example 3 to verify correctness?

**Aayush:** I have verified the correctness by printing the results for example 3.

**Interviewer:** In an interview, you need to be able to verbally trace — not just run the code. What are the time and space complexities? Any edge cases you haven't handled?

**Aayush:** Time complexity is O(2^n) since at each index we are choosing whether to break or not. Space complexity is O(n) for the recursion stack. I have not handled the case when the string is initially empty or there are no words in the wordDict.

**Interviewer:** Good on complexity. Can you optimize this? O(2^n) is not acceptable for n=300.

**Aayush:** We can use memoization to remember the boolean values for each combination of last and i.

**Interviewer:** What's the state space? How many unique (last, i) pairs, and what's the new TC?

**Aayush:** There are O(n²) state pairs so TC = O(n×N) and SC = O(n×n).

**Interviewer:** Close — you're missing the work done per state. substr and isPresent each cost something. What's the full TC?

**Aayush:** O(n × n × |wordDict| × |max(wordDict[i])|)

**Interviewer:** Correct for linear search. With unordered_set it simplifies to O(n³). Now can you think of a cleaner 1D DP formulation instead of 2D memoization?

**Aayush:** dp[i] = true if s[0:i] can be formed using the wordDict. At each i we would check if breaking is possible by iterating over wordDict.

**Interviewer:** What's the base case and transition?

**Aayush:** dp[0] = true. dp[i] = true if any word in wordDict ends at i and dp[i - wordDict[j].size()] is also true.

**Interviewer:** Perfect. Go ahead and code it up.

**Aayush:** [Submitted DP code — see Solution section below.]

**Interviewer:** Logic is correct. The set is redundant — you're already iterating over wordDict so str == word suffices. What's the TC and SC of this DP solution?

**Aayush:** TC = O(n × |wordDict| × |max(wordDict[i])|). SC = O(n).

**Interviewer:** Correct. Note your current time.

**Aayush:** 13:21

---

## Solution

**Aayush's Backtracking Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

bool isPresent(vector<string> &wordDict, string word)
{
    for(string str : wordDict)
    {
        if(str == word) return true;
    }
    return false;
}

bool solve(string s, vector<string> &wordDict, int last, int i)
{   
    int size = i - last;
    string str = s.substr(last+1, size);
    if(i == s.size())
    {
        return isPresent(wordDict, str);
    }
    
    bool ans = solve(s, wordDict, last, i+1);
    
    // breaking at i is possible
    if(isPresent(wordDict, str))
    {
       ans |= solve(s, wordDict, i, i+1); 
    }
    return ans;
}

int main() {
    vector<string> wordDict {"cats", "dog", "sand", "and", "cat"};
    string s = "catsandog";
    bool ans = solve(s, wordDict, -1, 0);
    cout << "ans is " << ans;
    return 0;
}
```

**Aayush's DP Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<string> wordDict {"cats", "dog", "sand", "and", "cat"};
    string s = "catsandog";
    int n = s.size();

    vector<bool> dp(n+1, 0);
    dp[0] = true;
    set<string> wordSet;
    for(string str : wordDict) wordSet.insert(str);
    
    for(int i = 1; i <= n; i++)
    {
        for(string word : wordDict)
        {   
            if(word.size() > i) continue;
            string str = s.substr(i - word.size(), word.size());
            if(wordSet.find(str) != wordSet.end())
            {
                dp[i] = dp[i] | dp[i - word.size()];
            }
        }
    }
    
    cout << "ans is " << dp[n];
    return 0;
}
```

**Optimal Solution (cleaner version of Aayush's DP):**
```cpp
bool wordBreak(string s, vector<string>& wordDict) {
    int n = s.size();
    unordered_set<string> wordSet(wordDict.begin(), wordDict.end());
    vector<bool> dp(n + 1, false);
    dp[0] = true;
    
    for(int i = 1; i <= n; i++) {
        for(string& word : wordDict) {
            int len = word.size();
            if(i >= len && dp[i - len] && s.substr(i - len, len) == word) {
                dp[i] = true;
                break;
            }
        }
    }
    return dp[n];
}
```

**Time Complexity:** O(n × |wordDict| × L) where L = max word length
**Space Complexity:** O(n) for dp array

---

## Feedback Given

### 1. Problem Understanding & Clarification (3.5/5)
Good instinct asking for constraints upfront. You also correctly identified the empty string and empty wordDict edge cases when prompted. However, you needed prompting to think about edge cases — in a real interview, you should surface these *before* coding, not after.

### 2. Approach & Thought Process (4/5)
Solid progression: backtracking → memoization → 1D DP. You correctly identified the state space for memoization (O(n²) pairs) and then independently derived the cleaner 1D DP formulation. The recurrence was correct on the first attempt.

### 3. Code Quality & Correctness (3/5)
- **Correct logic overall** — the backtracking and DP both work.
- **Redundant set**: you iterated over `wordDict` AND checked a set. Since you're already looping over words, `str == word` suffices.
- **String passed by value** in the recursive solution — unnecessary O(n) copy per call.
- **Did not trace through an example before claiming correctness.** "I verified by printing" is not the same as being able to walk an interviewer through your logic. This is a recurring pattern — you need to be able to dry-run your code verbally.

### 4. Complexity Analysis (4/5)
Got time and space complexity right for both solutions. Good that you proactively stated space complexity this time — that's an improvement. The per-state work breakdown (substr + lookup cost) was also correct.

### 5. Communication (3.5/5)
Generally clear, but terse. You needed prompting to elaborate on the recursion structure (the "or" ambiguity), the base case, and edge cases. In an interview, proactively explain your reasoning rather than waiting to be asked.

---

### Overall: 3.5/5 — Solid but needs polish

**Strengths:** Correct solution, good DP intuition, improved on space complexity analysis.

**Key area to fix:** Stop skipping the verbal trace before claiming correctness. Dry-run a small example on your own before saying "it works."
**Time Taken: 42 minutes**
