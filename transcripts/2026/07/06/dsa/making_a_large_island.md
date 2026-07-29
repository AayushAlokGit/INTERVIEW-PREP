# DSA Round Transcript
**Date:** 2026-07-06
**Start Time:** 10:00
**End Time:** 10:36
**Duration:** 36 minutes
**Problem:** Making a Large Island
**Topic:** Grid / Connected Components (Flood Fill)
**Difficulty:** Hard

---

## Problem Statement
You are given an `n x n` binary grid `grid`. You are allowed to change **at most one** `0` to a `1`.
Return the size of the **largest island** in `grid` after applying this operation.
An **island** is a 4-directionally connected group of `1`s.

**Example 1:**
```
Input:  grid = [[1,0],[0,1]]
Output: 3
Explanation: Change one 0 to 1 and connect two 1s, forming an island of area 3.
```
**Example 2:**
```
Input:  grid = [[1,1],[1,0]]
Output: 4
Explanation: Change the single 0 to a 1, joining all cells into one island of area 4.
```
**Example 3:**
```
Input:  grid = [[1,1],[1,1]]
Output: 4
Explanation: No 0 to flip, but the whole grid is already one island of area 4.
```

Constraints:
- `n == grid.length == grid[i].length`
- `1 <= n <= 500`
- `grid[i][j]` is `0` or `1`.

---

## Conversation Log

**Interviewer:** Presented the problem, constraints, and examples. Asked for clarifying questions before approach.

**Aayush:** What are the constraints?

**Interviewer:** Restated: square grid, `1 <= n <= 500` (up to 250,000 cells), cells are 0/1. Noted "at most one flip" (zero flips allowed) and return the cell count of the largest resulting island.

**Aayush:** Brute force is to flip each 0 and then get the largest island — O(n^4): n^2 for choosing each 0, n^2 for finding the largest island.

**Interviewer:** Correct baseline. Can you do better? What's wasteful about redoing the whole island scan every time?

**Aayush:** Recomputing island sizes is wasteful. Compute all island sizes on the initial grid once, then for each 0 check neighboring islands — if flipped, max size = neighboring island size + 1; take the maximum across all 0s.

**Interviewer:** Pushed on merge logic with a center-cell example touching four islands; asked whether "neighbor size + 1" captures multi-island merges.

**Aayush:** Perform disjoint set union (with rank) for the islands around the 0.

**Interviewer:** Asked him to concretely state what flipping the center produces and how to combine several islands.

**Aayush:** Noted that in the given example flipping the center gives size 1, since the corner 1s are only diagonally adjacent and don't contribute. (Correct — interviewer's example backfired.)

**Interviewer:** Acknowledged the correction. Gave two cleaner cases: Case A `[1,0,1]` (two distinct size-1 islands → 3) and Case B `[[1,1],[1,0]]` (both neighbors are the same size-3 island → 4). Asked how to compute so Case B isn't 7.

**Aayush:** Perform DSU of the 0 cell with all neighboring island cells.

**Interviewer:** Noted DSU union works and same-root union is a no-op, but raised two wrinkles: (1) only one 0 is flipped at a time, so live unions leave stale merges; (2) what prevents double-counting within a flip.

**Aayush:** Sum up the sizes of the 4-directional neighbor islands, then add 1, since flipping unites the individually separated islands.

**Interviewer:** Walked Case B: both neighbors report size 3, so raw sum gives 3+3+1 = 7, but the answer is 4. What must you track to add each distinct island only once?

**Aayush:** Addition only works for distinct neighboring islands. Need to assign each island a numeric id, track each island's size, and record which island id each 1 belongs to.

**Interviewer:** Confirmed the plan: (1) label pass — flood fill each island with a unique id (start at 2), record id→size, overwrite 1s with id; (2) flip pass — for each 0, gather the distinct neighbor ids in a set, sum their sizes, +1; (3) answer = max over candidates with fallback. Asked him to code it.

**Aayush:** Wrote the full C++ solution (below). Gave complexity: O(n^2) time (label + flip; the neighbor set is constant size), O(n^2) space (size vector + recursion stack up to n^2).

**Interviewer:** Logic is correct — traces to 7 on his example, dedup handles the merge, max_element init covers the no-zero case, complexity correct. Asked him to dry-run the smallest input `grid = [[1]]`, watching the size of `islandIdToSize` and every index written.

**Aayush:** `islandIdToSize` isn't of appropriate size for n=1; use an unordered_map instead of a vector to resolve it.

**Interviewer:** Confirmed: size is n*n+1 = 2 (indices 0,1) but islandId starts at 2, so `islandIdToSize[2]++` is out of bounds. unordered_map fixes it, or size the vector n*n+2. Solution complete and optimal. Moved to feedback.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

vector<vector<int>> dirs{{-1,0},{1,0},{0,1},{0,-1}};
void dfs(vector<vector<int>> &grid,int i,int j, int islandId,vector<int> &islandIdToSize)
{
    int n = grid.size();
    if(i<0 || j<0 || i>=n || j>=n) return;
    if(grid[i][j] != 1) return;
    grid[i][j] = islandId;
    islandIdToSize[islandId]++;
    for(auto dir:dirs)
    {
        int ni = i+dir[0];
        int nj = j+dir[1];
        dfs(grid,ni, nj, islandId, islandIdToSize);
    }
}
int main() {
    vector<vector<int>> grid {
        {1,0,1},
        {1,0,0},
        {1,1,1}
    };
    int n = grid.size();
    vector<int> islandIdToSize(n*n+1,0);

    // label all grid cells to the island Id to which they belong
    int islandId = 2;
    for(int i=0;i<n;i++)
        for(int j=0;j<n;j++)
            if(grid[i][j] == 1)
            {
                dfs(grid,i,j,islandId,islandIdToSize);
                islandId++;
            }

    int ans = *(max_element(islandIdToSize.begin(),islandIdToSize.end()));

    for(int i=0;i<n;i++)
        for(int j=0;j<n;j++)
            if(grid[i][j] == 0)
            {
                set<int> distinctNeighboringIslands;
                for(auto dir:dirs)
                {
                    int ni = i+dir[0];
                    int nj = j+dir[1];
                    if(ni>=0 && ni<n && nj>=0 && nj<n && grid[ni][nj] != 0)
                        distinctNeighboringIslands.insert(grid[ni][nj]);
                }
                int curSize = 0;
                for(int id: distinctNeighboringIslands)
                    curSize += islandIdToSize[id];
                curSize++;
                ans = max(ans, curSize);
            }

    cout<<"Ans is "<<ans;
    return 0;
}
```

**Fix identified during dry-run (n=1 boundary):** replace `vector<int> islandIdToSize(n*n+1,0)` with an `unordered_map<int,int>`, or size it `n*n+2`, because island ids start at 2 and can reach `1 + numIslands`.

**Optimal Solution:** Same as above (this is the canonical optimal approach). Time `O(n^2)`, space `O(n^2)`.

**Time Complexity:** O(n^2) — label pass O(n^2), flip pass O(n^2) with constant-size 4-neighbor set.
**Space Complexity:** O(n^2) — id→size storage plus worst-case recursion depth (snake-shaped island).

---

## Feedback Given

### Scoring

| Criterion | Score (out of 5) | Notes |
|---|---|---|
| Problem Understanding & Clarification | 3.5 | Asked for constraints (good — historically skipped). But accepted problem at face value otherwise; "at most one / zero-flip allowed" and "distinct vs total island" ambiguities surfaced only through prompting. |
| Approach & Thought Process | 4.5 | Clean progression: brute force → pre-compute sizes → recognized multi-island merge → distinct-id dedup. Reached for DSU, then pivoted well to the simpler label-and-map approach. |
| Code Quality & Correctness | 4 | Logically correct and clean first pass. Sensible id>=2 choice. One real boundary bug (n=1 sizing overflow) — found and fixed under a dry-run. |
| Complexity Analysis | 5 | Spot on. O(n^2) time with correct constant-neighbor reasoning; O(n^2) space accounting for both the size map and worst-case recursion depth. Marked improvement over usual "drops a factor" pattern. |
| Communication | 4 | Clear and steady. Good catch correcting the interviewer's flawed center-cell example. Could have volunteered the same-island double-count risk himself rather than being led via Case B. |

**Overall: 4.2 / 5 — Strong round.**

### What went well
- Asked for constraints unprompted — his #1 recurring gap; good to see it break.
- Efficient optimization path — didn't over-invest in DSU, adapted.
- Complexity analysis precise and complete, including recursion depth in space (a prior weak spot).
- Caught his own bug on the dry-run instead of defending the code.

### What to sharpen
1. Volunteer the tricky case before being asked. The distinct-island double-count (Case B) is the crux; he reached the right rule only after the counterexample was constructed for him.
2. Dry-run the boundary before declaring done. He had the right instinct on the n=1 overflow instantly — but only looked because prompted. Habit: run smallest input (single cell) and the no-op case (all 1s) proactively.
3. State edge cases out loud — e.g. "no zeros means the flip loop never runs, so max_element init carries the answer."
