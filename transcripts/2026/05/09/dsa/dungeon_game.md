# DSA Round Transcript
**Date:** 2026-05-09
**Start Time:** 17:38
**End Time:** 18:08
**Duration:** 30 minutes
**Problem:** Dungeon Game
**Topic:** Dynamic Programming (Grid DP, Reverse Traversal)
**Difficulty:** Hard

---

## Problem Statement

A knight is locked in the top-left cell of an `m x n` dungeon. The princess is in the bottom-right cell. The knight can only move **right** or **down** at each step.

Each cell has an integer — positive (gains health) or negative (loses health). The knight starts with some initial health and **dies if his health drops to 0 or below at any point**, even mid-path.

Return the **minimum initial health** the knight needs to rescue the princess.

**Example:**
```
dungeon = [
  [-2, -3,  3],
  [-5, -10, 1],
  [10,  30, -5]
]

Output: 7

Explanation:
Optimal path: right → right → down → down
Health: 7 → 5 → 2 → 5 → 6 → 1  (never drops to 0)
```

**Constraints:**
- `1 <= m, n <= 200`
- `-1000 <= dungeon[i][j] <= 1000`
- Knight must have at least 1 health at all times

---

## Conversation Log

**Interviewer:** Note your start time, then: any clarifying questions?

**Aayush:** 17:38. (No clarifying questions — jumped to approach.)

**Aayush:** dp[i][j] = min health needed to get to [m-1][n-1] from [i][j]. We can get dp[i][j] = min(dp[i+1][j], dp[i][j+1]) - dungeon[i][j]. Base case: dp[m-1][n-1] = 1 - dungeon[m-1][n-1]. Out of bounds returns INT_MAX.

**Interviewer:** Good direction. What if dungeon[m-1][n-1] is positive, say 5? What does 1-5 = -4 mean, and is that valid?

**Aayush:** Minimum initial health can be 0.

**Interviewer:** The knight must have at least 1 health at all times — even if the last cell gives +5, what's the minimum entering that cell?

**Aayush:** 1.

**Interviewer:** Exactly. So base case should be max(1, 1 - dungeon[m-1][n-1]). What about INT_MAX at boundary columns/rows?

**Aayush:** INT_MAX would be dropped since taking min.

**Interviewer:** Right. But after min(...) - dungeon[i][j], the result could be negative if dungeon[i][j] is large and positive. What do you do?

**Aayush:** Clamp to 1 using max(1, min(..) - dungeon[i][j]).

**Interviewer:** Perfect. Go ahead and code it.

**Aayush:** [wrote solution]

**Interviewer:** Time and space complexity? Edge cases?

**Aayush:** O(m*n) time, O(m*n) space. Edge cases: all positive numbers, all negative numbers.

**Interviewer:** Good. Can you think of a third edge case?

**Aayush:** A 1x1 grid — but the algorithm handles it via the base case.

**Interviewer:** Correct. Can you reduce space complexity?

**Aayush:** How to do this?

**Interviewer:** Think about what you need when computing dp[i][j] — you need dp[i+1][j] (row below) and dp[i][j+1] (same row, right). If you process one row at a time bottom to top, right to left — what's the minimum to store?

**Aayush:** The values of the next row. We could have a dp array holding the current row's values. After each row, dp becomes nextRow and we create a fresh dp.

**Interviewer:** Exactly — a single 1D array of size n, updated in-place right to left. Space drops to O(n).

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<vector<int>> dungeon{{-2, -3,  3}, {-5, -10, 1}, {10,  30, -5}};
    
    int m = dungeon.size();
    int n = dungeon[0].size();
    
    vector<vector<int>> dp(m, vector<int>(n));
    
    // dp[i][j] -> min health needed to reach m-1,n-1 from i,j
    dp[m-1][n-1] = max(1, 1 - dungeon[m-1][n-1]);
    
    for(int j = n-2; j >= 0; j--) dp[m-1][j] = max(1, dp[m-1][j+1] - dungeon[m-1][j]);
    
    for(int i = m-2; i >= 0; i--) dp[i][n-1] = max(1, dp[i+1][n-1] - dungeon[i][n-1]);
    
    for(int i = m-2; i >= 0; i--) {
        for(int j = n-2; j >= 0; j--) {
            dp[i][j] = max(1, min(dp[i+1][j], dp[i][j+1]) - dungeon[i][j]);
        }
    }
    cout << dp[0][0] << endl;
    return 0;
}
```

**Optimal Solution (space-optimized):**
```cpp
// Use a single 1D dp array of size n, process rows bottom to top, columns right to left
vector<int> dp(n, INT_MAX);
dp[n-1] = max(1, 1 - dungeon[m-1][n-1]);
for(int j = n-2; j >= 0; j--) dp[j] = max(1, dp[j+1] - dungeon[m-1][j]);

for(int i = m-2; i >= 0; i--) {
    dp[n-1] = max(1, dp[n-1] - dungeon[i][n-1]);
    for(int j = n-2; j >= 0; j--) {
        dp[j] = max(1, min(dp[j], dp[j+1]) - dungeon[i][j]);
    }
}
// dp[0] is the answer
```

**Time Complexity:** O(m * n)
**Space Complexity:** O(m * n) — reducible to O(n) with 1D optimization

---

## Feedback Given

### Problem Understanding & Clarification — 7/10
Jumped straight into the approach without asking clarifying questions — didn't confirm movement directions, health constraints, or whether the grid can be empty. Self-corrected the base case (0 → 1) when prompted, but that should have come from reading "at least 1 health at all times" more carefully upfront.

### Approach & Thought Process — 9/10
Excellent. Immediately identified the key insight — reverse DP (bottom-right to top-left) — which is the crux of this problem. Most candidates default to forward DP and get stuck. Clean recurrence definition.

### Code Quality & Correctness — 8/10
Code was clean, correct, and passed the example. Handled boundary rows/columns explicitly. The max(1, ...) clamp was applied consistently.

### Complexity Analysis — 8/10
Correct on time and space. Identified the O(n) space optimization when prompted — should aim to volunteer that without the hint.

### Communication — 7/10
Recurrence explanation was clear. Edge cases were shallow — only volunteered two, and the third (1x1) came after prompting. Should enumerate at least 3-4 edge cases unprompted.

### Overall: 39/50 — Strong performance, clean execution.

**Key takeaway:** Solved the hardest part (fill order) independently — that's the real skill here. Work on proactively volunteering optimizations and edge cases.

**Time Taken: 30 minutes**
