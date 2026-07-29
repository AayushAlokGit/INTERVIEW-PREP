# DSA Round Transcript
**Date:** 2026-06-03
**Start Time:** 20:25
**End Time:** 20:52
**Duration:** 27 minutes
**Problem:** Word Search
**Topic:** Backtracking / Grid DFS
**Difficulty:** Medium

---

## Problem Statement
Given an `m x n` grid of characters `board` and a string `word`, return `true` if `word` exists in the grid. The word is constructed from sequentially adjacent cells (horizontal/vertical neighbors); the same cell may not be used more than once within a single word.

Example:
```
board = [['A','B','C','E'],
         ['S','F','C','S'],
         ['A','D','E','E']]
word = "ABCCED" -> true
word = "SEE"    -> true
word = "ABCB"   -> false (B reused)
```

Constraints: m×n ≤ ~10^4; word length up to ~15 (can exceed cell count → false); letters only; cell reuse forbidden within a single match; word non-empty.

---

## Conversation Log

**Interviewer:** Presented Word Search; invited clarifying questions.
**Aayush:** Asked for constraints.
**Interviewer:** Provided constraints.
**Aayush:** Brute force via recursion `solve(row,col,i)` returns true if `word[i:]` found at (row,col). Base case `i==word.size()` → true; if `grid[row][col]!=word[i]` → false; else mark visited, recurse unvisited neighbors with i+1. TC O(4^|word|), SC O(|word|). Then claimed memoization on 3 states `(row,col,i)` brings TC to O(m·n·|word|), SC O(m·n).
**Interviewer:** Challenged the memoization validity (different paths) and the missing m·n factor in brute-force complexity.
**Aayush:** Conceded — visited set differs so memoization invalid; TC becomes O(m·n·4^|word|).
**Interviewer:** Asked him to code, watching visited management.
**Aayush:** Asked for a clearer example.
**Interviewer:** Gave 2×2 board "ABF" walkthrough + "ABS" false case.
**Aayush:** Wrote C++ solution (DFS with visited mark/restore, bounds + i==size + char/visited base cases, loop over 4 dirs, outer double loop over start cells).
**Interviewer:** Asked him to trace board=[['A']], word="A" line by line, focusing on base-case order.
**Aayush:** Identified the `i==size` check must be at the top before bounds/visited checks.
**Interviewer:** Confirmed fix; asked scope (which boards fail) and two optimizations.
**Aayush:** Scope: when last char matched but recursion lands out of bounds. Opt 1: string copied each call → pass const ref. Opt 2: early-return when ans true.
**Interviewer:** Refined scope to "only 1×1 board"; confirmed both optimizations. Wrapped up.

---

## Solution
**Aayush's Final Solution (as written, with noted bug):**
```cpp
bool solve(vector<vector<char>> &grid, int row, int col, int i, string word, vector<vector<int>> &visited) {
    if(row<0 || col<0 || row>=grid.size() || col>=grid[0].size()) return false;
    if(i==word.size()) return true;                       // BUG: should be at top, before bounds check
    if(grid[row][col] != word[i] || visited[row][col]) return false;
    visited[row][col] = 1;
    vector<vector<int>> dirs{{-1,0},{1,0},{0,1},{0,-1}};
    bool ans = false;
    for(auto dir:dirs) ans |= solve(grid,row+dir[0],col+dir[1],i+1,word,visited);
    visited[row][col] = 0;
    return ans;
}
// main: loop all (i,j), ans |= solve(grid,i,j,0,word,visited);
```

**Optimal Solution (corrected + optimized):**
```cpp
bool solve(vector<vector<char>>& g, int r, int c, int i, const string& word, vector<vector<int>>& vis) {
    if(i == word.size()) return true;                     // success check FIRST
    if(r<0 || c<0 || r>=g.size() || c>=g[0].size()) return false;
    if(vis[r][c] || g[r][c] != word[i]) return false;
    vis[r][c] = 1;
    static int dr[]={-1,1,0,0}, dc[]={0,0,1,-1};
    bool ans = false;
    for(int d=0; d<4 && !ans; d++)                        // early stop on success
        ans = solve(g, r+dr[d], c+dc[d], i+1, word, vis);
    vis[r][c] = 0;
    return ans;
}
```

**Time Complexity:** O(m·n·4^|word|) (tighter: O(m·n·3^|word|)) — corrected after prompting.
**Space Complexity:** O(|word|) recursion stack (+ O(m·n) visited grid).

---

## Feedback Given

**Time Taken: 27 minutes**

| Criterion | Score | Justification |
|---|---|---|
| Problem understanding & clarification | 4/5 | Asked constraints up front; asked for a clearer example. No gaps. |
| Approach & thought process | 3/5 | Correct DFS/backtracking instantly, but confidently proposed an invalid memoization (path-dependent state); corrected in one question. |
| Code quality & correctness | 3/5 | Clean visited mark/restore, but base-case ordering bug (1×1 returns false) found only when forced to trace. |
| Complexity analysis | 3/5 | Initial O(4^|word|) dropped the m·n start-cell factor; corrected when pushed. |
| Communication | 4/5 | Clear, self-corrects fast; slightly imprecise on bug scope. |

**One habit to hammer:** logic is consistently right, but points lost on (1) not tracing a degenerate/boundary input before submitting, and (2) stating complexity/optimization a half-step too loosely. Drills: run the degenerate input (empty / single / 1×1) before "done"; say the *why* whenever asserting "memoize" or "O(...)".
