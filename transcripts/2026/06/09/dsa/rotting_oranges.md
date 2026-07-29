# DSA Round Transcript
**Date:** 2026-06-09
**Start Time:** 9:43
**End Time:** 9:53
**Duration:** 10 minutes
**Problem:** Rotting Oranges
**Topic:** Multi-Source BFS (Grid)
**Difficulty:** Medium

---

## Problem Statement
Given an `m x n` grid where each cell is `0` (empty), `1` (fresh orange), or `2` (rotten orange). Every minute, any fresh orange 4-directionally adjacent to a rotten orange becomes rotten. Return the minimum number of minutes until no cell has a fresh orange, or `-1` if impossible.

**Example 1:**
```
grid = [[2,1,1],[1,1,0],[0,1,1]]  ->  4
```
**Example 2:**
```
grid = [[2,1,1],[0,1,1],[1,0,1]]  ->  -1  (orange at (2,0) never reachable)
```
**Example 3:**
```
grid = [[0,2]]  ->  0  (no fresh oranges)
```

**Constraints:**
- `m == grid.length`, `n == grid[i].length`
- `1 <= m, n <= 10`
- `grid[i][j]` is `0`, `1`, or `2`

---

## Conversation Log

**Interviewer:** Presented the problem; asked for clarifying questions / approach.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints (1 <= m, n <= 10; cells 0/1/2). Asked for approach.

**Aayush:** Multi-source BFS. Put all rotten oranges in the queue at time 0; as time progresses, neighbors of current rotten oranges become rotten and get added to the queue. The max level of the BFS is the required time. If a fresh orange remains, it's impossible. TC O(mn) since BFS visits each cell at most once; SC O(mn) for the queue.

**Interviewer:** Good, and good that he volunteered complexity. Asked: (1) how exactly to count minutes avoiding the off-by-one (Example 3 = 0, not 1), and (2) how to detect impossibility — re-scan or track a counter?

**Aayush:** Initial level is 0; keep track of total fresh oranges and decrement the count during BFS; if remaining after BFS, impossible.

**Interviewer:** Clean. Asked him to code it.

**Aayush:** (submitted C++ solution below).

**Interviewer:** Probed two points: (1) if you drop the `&& fresh > 0` guard and just loop `while(!q.empty())`, does the answer change and why? (2) Edge cases — all 0s, all 2s, single fresh orange with no rotten?

**Aayush:** (1) Even without fresh > 0 the BFS works. (2) All 0s — should it return -1? Currently returns 0. All 2s — returns 0. Single fresh orange — BFS doesn't execute, returns -1.

**Interviewer:** Challenged #1 — asked him to trace `[[2,1]]` without the guard, step by step. Clarified #2: problem asks minutes until no fresh oranges; if none exist, 0 minutes is required.

**Aayush:** It will cause an off-by-one for minutes. All-0s should be 0.

**Interviewer:** Correct on both. Without the guard, the final wave processes already-rotten oranges, finds nothing fresh, but still increments minutes — over-counting by one. All-0s correctly returns 0. Solution is correct and optimal.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <iostream>
#include <vector>
#include <queue>
using namespace std;

int orangesRotting(vector<vector<int>>& grid) {
    int rows = grid.size();
    int cols = grid[0].size();

    queue<pair<int, int>> q;
    int fresh = 0;

    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) {
            if (grid[i][j] == 2) {
                q.push({i, j});
            } else if (grid[i][j] == 1) {
                fresh++;
            }
        }
    }

    if (fresh == 0) return 0;

    int minutes = 0;
    vector<pair<int, int>> directions = {
        {1, 0}, {-1, 0}, {0, 1}, {0, -1}
    };

    while (!q.empty() && fresh > 0) {
        int size = q.size();

        for (int i = 0; i < size; i++) {
            auto [x, y] = q.front();
            q.pop();

            for (auto [dx, dy] : directions) {
                int nx = x + dx;
                int ny = y + dy;

                if (nx >= 0 && nx < rows &&
                    ny >= 0 && ny < cols &&
                    grid[nx][ny] == 1) {

                    grid[nx][ny] = 2; // Rot it
                    fresh--;
                    q.push({nx, ny});
                }
            }
        }

        minutes++;
    }

    return (fresh == 0) ? minutes : -1;
}
```
**Optimal Solution (if different):** Same — this is optimal.

**Time Complexity:** O(m*n)
**Space Complexity:** O(m*n)

---

## Feedback Given

**Problem Understanding & Clarification — 4.5/5**
Asked for constraints up front; pinned down the minute-counting off-by-one during approach discussion before coding.

**Approach & Thought Process — 5/5**
Multi-source BFS with fresh counter — textbook correct. Volunteered complexity unprompted (the gap flagged last round). Nice adjustment.

**Code Quality & Correctness — 5/5**
Correct first pass including the subtle `&& fresh > 0` guard and the no-fresh early return. No bugs.

**Complexity Analysis — 5/5**
O(mn) time and space, stated up front and accurate. Strong improvement over last round's vague-O(n).

**Communication — 4/5**
Volunteered three edge cases on his own (addresses the recurring shallow-edge-coverage gap). Deduction: first instinct on the guard question was to assert "BFS works without fresh > 0" — wrong; only saw the off-by-one when forced to trace `[[2,1]]`. Recurring pattern: asserts in head instead of tracing the smallest breaking case when challenged on correctness. Knowledge was there (code was correct) but the verbal answer was wrong.

**Overall:** Excellent round. Correct code, two of last round's gaps closed (volunteered complexity + edge cases). Remaining habit: trace before asserting when challenged on correctness.

**Time Taken: 10 minutes**
