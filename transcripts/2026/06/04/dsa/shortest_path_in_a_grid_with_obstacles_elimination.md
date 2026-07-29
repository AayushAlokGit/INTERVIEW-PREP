# DSA Round Transcript
**Date:** 2026-06-04
**Start Time:** 10:53
**End Time:** 11:24
**Duration:** 31 minutes
**Problem:** Shortest Path in a Grid with Obstacles Elimination
**Topic:** Graphs / BFS (state-augmented)
**Difficulty:** Hard

---

## Problem Statement
Given an `m x n` grid where each cell is `0` (empty) or `1` (obstacle), move up/down/left/right between empty cells in one step. Return the minimum steps from `(0,0)` to `(m-1,n-1)` given you can eliminate at most `k` obstacles. Return `-1` if impossible.

**Example 1:**
```
grid = [[0,0,0],[1,1,0],[0,0,0],[0,1,1],[0,0,0]], k = 1  →  6
```
**Example 2:**
```
grid = [[0,1,1],[1,1,1],[1,0,0]], k = 1  →  -1
```

**Constraints:**
- `1 <= m, n <= 40`
- `1 <= k <= m*n`
- `grid[i][j]` in {0,1}; `grid[0][0] == grid[m-1][n-1] == 0`

---

## Conversation Log

**Interviewer:** Presented the problem; asked for clarifying questions and approach.

**Aayush:** "what are the constraints?"

**Interviewer:** Gave constraints (grid ≤ 40×40, k ≤ m*n, endpoints empty).

**Aayush:** Proposed a recursive brute force `solve(i,j,k)` → min steps to reach (m-1,n-1) from (i,j) using ≤ k eliminations. Base case at destination = 0; if k==0 and cell is obstacle return INF; decrement k by grid[i][j]; recurse into 4 neighbors, take min of non-INF results.

**Interviewer:** Pushed on cycles — what stops infinite back-and-forth in 4-directional recursion? And is DFS the right tool for shortest path with unit edges?

**Aayush:** Suggested a visited set. Then offered an alternative: "get the shortest path src→dest via BFS, count obstacles on it; if cnt > k not possible, else answer is the path length. Recover the path via parent mapping."

**Interviewer:** Showed the flaw with a counterexample — Route A: 5 steps / 2 obstacles vs Route B: 9 steps / 0 obstacles, k=1. Globally shortest path (A) is infeasible; answer is B (9). Obstacle count is a constraint baked into reachability, not a post-hoc check. Asked: is `(i,j)` alone a sufficient state, given two paths can reach a cell having used different eliminations?

**Aayush:** "The state also needs to carry the number of eliminations that can still be done; if k becomes 0 and we hit an obstacle, don't push that state. This way we consider eliminations."

**Interviewer:** Confirmed state = (row, col, remainingK). Asked (1) what the visited structure marks, and (2) what BFS level-order guarantees about the first time the destination is popped.

**Aayush:** (1) "keep elimination state in visited too." (2) "first time we reach the destination is the shortest path considering eliminations allowed."

**Interviewer:** Confirmed; asked him to code.

**Aayush:** Submitted the C++ BFS solution (below). Stated "TC O(m*n*k), SC O(m*n*k)."

**Interviewer:** Confirmed correct. Pressed on complexity — `visited` is a `std::set<vector<int>>`; what's the per-op cost and what backs it?

**Aayush:** "TC is m*n*k*log(m*n*k)."

**Interviewer:** Confirmed (set = balanced BST, O(log N) ops). Asked (1) how to remove the log factor and (2) whether (i,j,remK) is the most compact state — is arriving again with ≤ remaining budget ever useful?

**Aayush:** "store remK[row][col] and update if we arrive at same cell with lesser remK ... reduces SC to O(n)." (Interrupted.)

**Interviewer:** Corrected the inverted comparison — keep the arrival with MORE budget, prune the one with ≤. Space of the 2D maxRemK grid is O(m*n) (not O(n)); a 2D array / unordered_set gives O(1) lookups, removing the log factor → true O(m*n*k). Noted the Manhattan shortcut: if k ≥ m+n-2, answer = m+n-2. Asked for end time.

**Aayush:** "11:24."

---

## Solution
**Aayush's Final Solution:**
```cpp
int shortestPath(vector<vector<int>>& grid, int k) {
    int m = grid.size(), n = grid[0].size();
    queue<vector<int>> q;
    set<vector<int>> visited;
    q.push({0, 0, k, 0}); // row, col, remainingK, distance
    visited.insert({0, 0, k});
    vector<pair<int,int>> dirs = {{1,0},{-1,0},{0,1},{0,-1}};

    while (!q.empty()) {
        auto curr = q.front(); q.pop();
        int row = curr[0], col = curr[1], remK = curr[2], dist = curr[3];
        if (row == m - 1 && col == n - 1) return dist;
        for (auto &[dr, dc] : dirs) {
            int nr = row + dr, nc = col + dc;
            if (nr < 0 || nr >= m || nc < 0 || nc >= n) continue;
            int newK = remK;
            if (grid[nr][nc] == 1) newK--;
            if (newK < 0) continue;
            vector<int> state = {nr, nc, newK};
            if (!visited.count(state)) {
                visited.insert(state);
                q.push({nr, nc, newK, dist + 1});
            }
        }
    }
    return -1;
}
```

**Optimizations discussed:**
- Replace `std::set` (O(log N) per op) with `unordered_set` or a 3D/2D array → true O(m·n·k) time.
- Compress visited to `maxRemK[i][j]` = max budget ever seen on reaching (i,j); skip an arrival if `remK <= maxRemK[i][j]` (more budget is strictly better). Visited footprint O(m·n).
- Early shortcut: if `k >= m + n - 2`, return `m + n - 2` (Manhattan distance).

**Time Complexity:** As written O(m·n·k·log(m·n·k)) due to `std::set`; O(m·n·k) with a hash/array visited. (Aayush initially said O(m·n·k), corrected to the log version when prompted.)
**Space Complexity:** O(m·n·k) for queue + visited; visited reducible to O(m·n) with the maxRemK trick.

---

## Feedback Given

### What you did well
- Constraints-first across all three rounds.
- Correct, clean state-augmented BFS on the first write.
- Recovered from a flawed "shortest path then count obstacles" idea after one counterexample — saw k as a reachability constraint.
- Self-corrected complexity to include the std::set log factor.

### What to sharpen
1. Problem-specific coupling: a constraint that can force a longer answer belongs inside the search state, not a post-check.
2. Complexity precision (3rd round running): `std::set` is a BST (O(log N) ops) — ask "ordered or hashed?" before claiming O(1) lookups.
3. Volunteer optimizations unprompted (unordered_set, maxRemK pruning, Manhattan shortcut).
4. Mind the direction of a pruning inequality — keep MORE budget, prune less.

### Scores (out of 5)
| Criterion | Score | Note |
|---|---|---|
| Problem understanding & clarification | 4.5 | Constraints upfront |
| Approach & thought process | 3.5 | Flawed decoupled idea first; recovered to state-augmented BFS |
| Code quality & correctness | 4.5 | Correct, clean, first try |
| Complexity analysis | 3.5 | Missed std::set log factor; corrected when asked |
| Communication | 4 | Engaged; inverted prune inequality; optimizations only when prompted |

**Overall: 4 / 5** — strong solve on a hard graph problem. Session pattern: algorithms right, complexity statements loose. Tighten the Big-O reflex for a clean senior bar.

**Time Taken: 31 minutes**
