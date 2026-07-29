# DSA Round Transcript
**Date:** 2026-06-08
**Start Time:** 9:53
**End Time:** 10:22
**Duration:** 29 minutes
**Problem:** Pacific Atlantic Water Flow
**Topic:** Graphs / Multi-source BFS / Grid
**Difficulty:** Medium

---

## Problem Statement
An `m x n` island borders the Pacific Ocean (top and left edges) and the Atlantic Ocean (right and bottom edges). Given matrix `heights`, water flows from a cell to a 4-directional neighbor if the neighbor's height is `<=` the current cell's height. Return all coordinates `(r, c)` from which water can reach **both** oceans.

**Example:**
```
heights = [
  [1, 2, 2, 3, 5],
  [3, 2, 3, 4, 4],
  [2, 4, 5, 3, 1],
  [6, 7, 1, 4, 5],
  [5, 1, 1, 2, 4]
]
Expected: [[0,4],[1,3],[1,4],[2,2],[3,0],[3,1],[4,0]]
```

**Constraints:**
- m == heights.length, n == heights[r].length
- 1 <= m, n <= 200
- 0 <= heights[r][c] <= 10^5

---

## Conversation Log

**Interviewer:** Presented the problem; asked for clarifying questions.

**Aayush:** "what are the constraints?"

**Interviewer:** Provided constraints (grid up to 200x200, heights 0..1e5, can repeat).

**Aayush:** Proposed reverse BFS inward from each ocean's border cells. Maintain a BFS queue and two boolean matrices `pacificReachable` / `atlanticReachable`. Seed the queue with the relevant border cells (top+left for Pacific). Pop a cell, visit 4-directional neighbors whose height `>=` current cell's height and not yet visited, enqueue them. Repeat for the Atlantic. Combine cells reachable by both.

**Interviewer:** Validated the reverse-BFS insight. Probed: justify the `>=` neighbor condition under reverse traversal; and is there a risk of the same cell being queued multiple times if marked on pop?

**Aayush:** "because we are essentially moving in the backwards direction of water." And: "Mark visited when pushing to queue to fully support the invariant that the queue holds coordinates which are reachable."

**Interviewer:** Confirmed both. Asked him to code in C++ including the combine step.

**Aayush:** Requested boilerplate in `int main` format.

**Interviewer:** Provided skeleton.

**Aayush:** Submitted full solution (below) and stated "Time complexity is O(m*n) and SC is O(m*n)."

**Interviewer:** Compiled and ran the code against the example — output matched expected exactly. Confirmed complexity. Noted the `dirs` vector being rebuilt inside the loop as a cosmetic nit.

---

## Solution

**Aayush's Final Solution (correct, verified by execution):**
```cpp
vector<vector<int>> pacificAtlantic(vector<vector<int>>& heights) {
    int m = heights.size();
    int n = heights[0].size();
    vector<vector<int>> pacificVis(m, vector<int>(n, 0));
    vector<vector<int>> atlanticVis(m, vector<int>(n, 0));
    queue<pair<int,int>> bfsQ;

    // Pacific: seed top + left borders
    bfsQ.push({0,0}); pacificVis[0][0] = 1;
    for (int i = 1; i < m; i++) { bfsQ.push({i,0}); pacificVis[i][0] = 1; }
    for (int j = 1; j < n; j++) { bfsQ.push({0,j}); pacificVis[0][j] = 1; }
    while (!bfsQ.empty()) {
        auto [r,c] = bfsQ.front(); bfsQ.pop();
        vector<vector<int>> dirs{{-1,0},{1,0},{0,1},{0,-1}};
        for (auto dir : dirs) {
            int nr = r + dir[0], nc = c + dir[1];
            if (nr>=0 && nr<m && nc>=0 && nc<n && pacificVis[nr][nc]!=1 && heights[nr][nc] >= heights[r][c]) {
                pacificVis[nr][nc] = 1; bfsQ.push({nr,nc});
            }
        }
    }

    // Atlantic: seed bottom + right borders
    bfsQ.push({m-1,n-1}); atlanticVis[m-1][n-1] = 1;
    for (int i = 0; i < m-1; i++) { bfsQ.push({i,n-1}); atlanticVis[i][n-1] = 1; }
    for (int j = 0; j < n-1; j++) { bfsQ.push({m-1,j}); atlanticVis[m-1][j] = 1; }
    while (!bfsQ.empty()) {
        auto [r,c] = bfsQ.front(); bfsQ.pop();
        vector<vector<int>> dirs{{-1,0},{1,0},{0,1},{0,-1}};
        for (auto dir : dirs) {
            int nr = r + dir[0], nc = c + dir[1];
            if (nr>=0 && nr<m && nc>=0 && nc<n && atlanticVis[nr][nc]!=1 && heights[nr][nc] >= heights[r][c]) {
                atlanticVis[nr][nc] = 1; bfsQ.push({nr,nc});
            }
        }
    }

    vector<vector<int>> ans;
    for (int i = 0; i < m; i++)
        for (int j = 0; j < n; j++)
            if (pacificVis[i][j] && atlanticVis[i][j]) ans.push_back({i,j});
    return ans;
}
```
Verified output: `[0,4] [1,3] [1,4] [2,2] [3,0] [3,1] [4,0]` — matches expected.

**Optimal Solution:** Same as above (this IS the optimal approach). Minor cleanup: hoist `dirs` out of the BFS loop / make it `static const`.

**Time Complexity:** O(m·n) — each cell enqueued at most once per ocean, 4 neighbor checks each.
**Space Complexity:** O(m·n) — two visited matrices plus the queue.

---

## Feedback Given

This was a clean pass and a clear step up from the prior round — correct code on first submission, verified by execution.

**What went well**
- Chose the reverse-BFS-from-borders insight immediately (avoided the naive O((m·n)^2) per-cell search) — directly improving on the "defaults to generic over structure-exploiting" weakness.
- Justified the inverted `>=` comparison by reasoning about reverse water flow, not memorization.
- Caught the mark-on-enqueue subtlety himself and articulated the queue invariant.
- Correct, clean code first try; compiled and produced exact expected output.
- Volunteered complexity without prompting and got both time and space right — the biggest improvement vs. the recurring vague-O(n) weakness.

**What to keep sharpening**
- Still didn't dry-run before declaring done — the interviewer ran it. Code was correct so it cost nothing, but the self-verification habit isn't yet his own reflex.
- Cosmetic: `dirs` rebuilt inside every loop iteration; hoist it out.

**Scoring (out of 5)**
| Category | Score |
|---|---|
| Problem understanding & clarification | 4.5 |
| Approach & thought process | 5 |
| Code quality & correctness | 4.5 |
| Complexity analysis | 5 |
| Communication | 4.5 |

**Overall: ~4.7/5** — strong senior-level performance. The contrast with the prior round is the lesson: reasoning from structure and owning the invariant yields correct code the first time.

**Time Taken: 29 minutes**
