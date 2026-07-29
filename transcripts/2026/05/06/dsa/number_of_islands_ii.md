# DSA Round Transcript
**Date:** 2026-05-06
**Start Time:** 12:56
**End Time:** 13:36
**Duration:** 40 minutes
**Problem:** Number of Islands II
**Topic:** Disjoint Set Union (DSU)
**Difficulty:** Hard

---

## Problem Statement
Given an empty 2D binary grid of size m x n (all water initially), perform a series of `addLand` operations defined by `positions[i] = [ri, ci]`. After each operation, return the number of islands.

An island is a maximal group of land cells connected horizontally or vertically.

Constraints:
- 1 <= m, n, positions.length <= 10^4
- 1 <= m * n <= 10^4
- Positions may contain duplicates.

Examples:
- m=3, n=3, positions=[[0,0],[0,1],[1,2],[2,1]] → [1,1,2,3]
- m=1, n=1, positions=[[0,0]] → [1]
- m=3, n=3, positions=[[0,0],[0,0],[1,1]] → [1,1,2]

---

## Conversation Log

**Interviewer:** Presented problem. Asked for start time and clarifying questions.
**Aayush:** 12:56. What are the constraints?
**Interviewer:** Provided constraints, including duplicates.
**Aayush:** Brute force: compute number of islands after each operation using BFS — O(positions * m * n).
**Interviewer:** Confirmed and pushed for better. Hinted at "merge + count groups" data structure.
**Aayush:** DSU. Each addLand checks 4 neighbors and unions.
**Interviewer:** Asked for more detail on representation, count maintenance, and duplicate handling.
**Aayush:** Asked what happens when addLand performed consecutively on same cell.
**Interviewer:** Confirmed island count should not change for duplicates.
**Aayush:** Described approach: parent array, parent[cell]++ on add, union with neighbors having parent > 0, answer = count of unique values > 0.
**Interviewer:** Pointed out parent[cell]++ is not DSU; counting unique parents is O(mn) per step. Provided correct DSU pattern: parent[]=-1 sentinel, separate count, union returns bool, decrement only on successful merge.
**Aayush:** Asked for inputs again.
**Interviewer:** Restated.
**Aayush:** First submission — 6 bugs: `if(parent[num]=num)`, void return type, `row`/`col` typos, `conut--`, broken bounds check (allowed wraparound), unconditional `count--`.
**Interviewer:** Walked through each bug with specific failing input for the bounds check (r=1, c=0 going left → newCell = n-1 wraps to row 0).
**Aayush:** Second submission — 5 bugs unfixed: same `=` vs `==`, `union` is C++ reserved keyword, `True` instead of `true`, bounds check still flat-index only, `shoulDelete` captured but ignored.
**Interviewer:** Provided corrected reference solution. Asked for complexity.
**Aayush:** O(positions * log(mn)) time, O(mn) space.
**Interviewer:** Refined: with path compression + union by rank, it's O(k * α(mn)) ≈ O(k). Asked for end time.
**Aayush:** 13:36.

---

## Solution
**Aayush's Final Solution (had unfixed bugs — see feedback):**
```cpp
int findParent(vector<int>& parent, int num) {
    if (parent[num] = num) return num;  // BUG: = should be ==
    return parent[num] = findParent(parent, parent[num]);
}
bool union(vector<int>& parent, int n1, int n2) {  // BUG: union is reserved
    int pn1 = findParent(parent, n1);
    int pn2 = findParent(parent, n2);
    if (pn1 == pn2) return false;
    parent[n1] = pn2;
    return True;  // BUG: True -> true
}
vector<int> numIslands2(int m, int n, vector<vector<int>>& positions) {
    vector<int> parent(m*n, -1);
    int count = 0;
    vector<int> ans;
    for (int i = 0; i < positions.size(); i++) {
        auto vec = positions[i];
        int r = vec[0], c = vec[1];
        int cell = row*n + col;  // BUG: row/col -> r/c
        if (parent[cell] == -1) {
            count++;
            parent[cell] = cell;
            vector<vector<int>> dirs{{-1,0},{1,0},{0,1},{0,-1}};
            for (auto dir : dirs) {
                int newR = r+dir[0], newC = c+dir[1];
                int newCell = newR*n + newC;
                if (newCell >= 0 && newCell < m*n && parent[newCell] != -1) {  // BUG: bounds check allows wraparound
                    bool shoulDelete = union(parent, newCell, cell);
                    count--;  // BUG: unconditional decrement
                }
            }
        }
        ans.push_back(count);
    }
    return ans;
}
```

**Optimal Solution:**
```cpp
int findParent(vector<int>& parent, int num) {
    if (parent[num] == num) return num;
    return parent[num] = findParent(parent, parent[num]);
}

bool unionSets(vector<int>& parent, vector<int>& rank_, int n1, int n2) {
    int p1 = findParent(parent, n1);
    int p2 = findParent(parent, n2);
    if (p1 == p2) return false;
    if (rank_[p1] < rank_[p2]) swap(p1, p2);
    parent[p2] = p1;
    if (rank_[p1] == rank_[p2]) rank_[p1]++;
    return true;
}

vector<int> numIslands2(int m, int n, vector<vector<int>>& positions) {
    vector<int> parent(m*n, -1);
    vector<int> rank_(m*n, 0);
    int count = 0;
    vector<int> ans;
    vector<vector<int>> dirs{{-1,0},{1,0},{0,1},{0,-1}};

    for (auto& pos : positions) {
        int r = pos[0], c = pos[1];
        int cell = r*n + c;
        if (parent[cell] == -1) {
            parent[cell] = cell;
            count++;
            for (auto& d : dirs) {
                int nr = r + d[0], nc = c + d[1];
                if (nr >= 0 && nr < m && nc >= 0 && nc < n) {
                    int neighbor = nr*n + nc;
                    if (parent[neighbor] != -1 && unionSets(parent, rank_, neighbor, cell)) {
                        count--;
                    }
                }
            }
        }
        ans.push_back(count);
    }
    return ans;
}
```

**Time Complexity:** O(k * α(mn)) ≈ O(k) with PC + union by rank; Aayush stated O(k * log(mn)).
**Space Complexity:** O(mn)

---

## Feedback Given

**Problem Understanding & Clarification: 4.5/5**
- Asked for constraints upfront — good correction from previous round.
- Caught duplicate-position edge case proactively.

**Approach & Thought Process: 4/5**
- Identified brute force quickly with correct complexity.
- Made the leap to DSU on prompting.
- Stumbled on running-count maintenance (suggested counting unique parents — would defeat optimization).

**Code Quality & Correctness: 1.5/5**
- First submission had 6 distinct bugs (compile errors + logic bugs).
- Second submission still had 5 of those bugs unfixed despite explicit walkthrough — including the bounds check after a specific failing input was provided.
- Strongest signal yet of recurring "doesn't trace edge cases / doesn't re-verify after fixing" weakness.

**Complexity Analysis: 4/5**
- Stated time and space proactively. Imprecise on DSU constant: with both optimizations it's O(α(mn)).

**Communication: 3.5/5**
- Clear initial approach. Did not absorb bug-list feedback carefully — re-submitted with most bugs unfixed.

**Overall: 17.5/25**
**Time Taken: 40 minutes**

**Key takeaways:**
1. When given a list of bugs, fix each one explicitly and re-trace line by line.
2. Bounds checking on grids: always check row and col separately, never flat-index alone.
3. Memorize DSU template with path compression + union by rank — should be ~2 min to write cleanly.
4. C++ reserved keywords: `union`, `class`, `template`, `new`, `delete` — don't use as identifiers.
