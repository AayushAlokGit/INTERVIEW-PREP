# DSA Round Transcript
**Date:** 2026-05-19
**Start Time:** 11:17
**End Time:** 11:32
**Duration:** 15 minutes
**Problem:** Edit Distance (LC 72)
**Topic:** Dynamic Programming
**Difficulty:** Hard

---

## Problem Statement

Given two strings `word1` and `word2`, return the minimum number of operations required to convert `word1` to `word2`.

Operations allowed: Insert, Delete, Replace.

**Example 1:** word1 = "horse", word2 = "ros" → 3
**Example 2:** word1 = "intention", word2 = "execution" → 5

**Constraints:** 0 <= word1.length, word2.length <= 500, lowercase English.

---

## Conversation Log

**Interviewer:** Note current time, share, and any clarifying questions.

**Aayush:** 11:17

**Aayush:** We can use DP where dp[i][j] = min operations to convert word1[0:i] to word2[0:j]. dp[0][j]=j, dp[i][0]=i. dp[i][j] = min(dp[i-1][j-1]+1 replace, dp[i-1][j]+1 delete, dp[i][j-1]+1 insert).

**Interviewer:** Missing one case — what if word1[i-1] == word2[j-1]?

**Aayush:** If they are equal then dp[i][j] = dp[i-1][j-1].

**Interviewer:** Correct. Code it up.

**Aayush:** [submits clean 2D DP solution]

**Interviewer:** TC, SC, and can you optimize space?

**Aayush:** TC O(n*m), SC O(n*m). No I don't think space can be optimized further.

**Interviewer:** dp[i][j] only depends on dp[i-1][j-1], dp[i-1][j], dp[i][j-1]. Which rows are you actually using?

**Aayush:** I'm using only current row and last row.

**Interviewer:** So optimized SC? And can you go further than 2 rows?

**Aayush:** Optimized SC is O(n).

**Interviewer:** Row size is m+1 so it's O(m), or O(min(n,m)) if you choose. And yes, you can use one row + a temp variable for the diagonal.

**Aayush:** Right sorry O(m) and no.

**Interviewer:** Edge cases?

**Aayush:** Empty w1 and w2. Equal w1 and w2.

**Interviewer:** Good — both handled by base cases.

**Aayush:** 11:32

---

## Solution

**Aayush's Final Solution:**
```cpp
int minDistance(string word1, string word2) {
    int n = word1.size(), m = word2.size();
    vector<vector<int>> dp(n + 1, vector<int>(m + 1));
    for(int i = 0; i <= n; i++) dp[i][0] = i;
    for(int j = 0; j <= m; j++) dp[0][j] = j;
    for(int i = 1; i <= n; i++) {
        for(int j = 1; j <= m; j++) {
            if(word1[i-1] == word2[j-1]) dp[i][j] = dp[i-1][j-1];
            else dp[i][j] = 1 + min({dp[i-1][j], dp[i][j-1], dp[i-1][j-1]});
        }
    }
    return dp[n][m];
}
```

**Time Complexity:** O(n*m)
**Space Complexity:** O(n*m) → optimizable to O(min(n,m))

---

## Feedback Given

| Category | Score (/5) | Notes |
|---|---|---|
| Problem Understanding & Clarification | 3 | No clarifying questions. |
| Approach & Thought Process | 4 | Recognized DP immediately. Missed equal-char case; caught with one nudge. |
| Code Quality & Correctness | 5 | Clean, correct first write. |
| Complexity Analysis | 3.5 | Said O(n) for optimized SC; should be O(m). Flatly said "can't optimize" before prompted. |
| Communication | 3.5 | Reactive — didn't volunteer space opt or equality case. |

**Overall: ~3.8/5**

### Key Takeaways
1. Volunteer space optimizations unprompted (recurring weakness).
2. Be precise about dimensions in complexity.
3. Push edge case coverage deeper.

**Time Taken: 15 minutes**
