# DSA Round Transcript
**Date:** 2026-05-18
**Start Time:** 10:30
**End Time:** 11:11
**Duration:** 41 minutes
**Problem:** Kth Smallest Element in a Sorted Matrix
**Topic:** Binary Search on Answer / Sorted Matrix
**Difficulty:** Medium-Hard

---

## Problem Statement
Given an `n x n` matrix where each row and each column is sorted ascending, return the **k-th smallest element** (in sorted order, not k-th distinct).

Memory must be better than O(n²).

### Example 1:
```
matrix = [[1, 5, 9], [10, 11, 13], [12, 13, 15]], k = 8 → 13
```
Sorted: [1,5,9,10,11,12,13,13,15]; 8th = 13.

### Constraints:
- 1 ≤ n ≤ 300
- -10⁹ ≤ matrix[i][j] ≤ 10⁹
- 1 ≤ k ≤ n²

---

## Conversation Log

**Interviewer:** Presented problem. Asked for time/clarifications/approach.

**Aayush:** Proposed max-heap of size k, iterate all elements, push and pop if size > k.

**Interviewer:** Correct but ignores sorted structure. Pushed on (1) merge-sorted-sequences idea, (2) binary search on answer using "how many ≤ x" predicate.

**Aayush:** Picked binary search on answer. Predicate: count(≤x). If count ≤ k, candidate; else increase x. Per-row count via `upper_bound` → O(n log n) per check; total O(n log² n).

**Interviewer:** Pressed on precision of condition. Asked: smallest x with count(≤x) ___ k? Which direction to move? Why does it converge to a matrix element? Also offered O(n) staircase trick.

**Aayush:** Refined: smallest x with `count(≤x) >= k`. When count < k, move right; else move left.

**Interviewer:** Explained O(n) staircase: start bottom-left, if `≤ x` add `row+1` and move right, else move up. Asked about duplicates.

**Interviewer:** Confirmed duplicates allowed and counted.

**Aayush:** Coded C++ solution with the staircase count and binary search.

**Interviewer:** Traced [[1,5,9],[10,11,13],[13,13,13]] with k=5 → 11. Correct.

**Interviewer:** Pushed on (a) O(n²) bounds scan inefficiency, (b) total complexity, (c) convergence proof.

**Aayush:** lo = matrix[0][0], hi = matrix[n-1][n-1]. Complexity O(n log(max−min)), O(1) space. Convergence: count function only changes at matrix elements.

**Interviewer:** Sharpened proof: if x not in matrix, count(≤x) = count(≤x-1), so x-1 also satisfies predicate — contradicts minimality.

---

## Solution

**Aayush's Final Solution (C++):**
```cpp
#include <bits/stdc++.h>
using namespace std;

int count(vector<vector<int>> &matrix, int x) {
    int cnt = 0;
    int row = matrix.size() - 1;
    int col = 0;
    while (row >= 0 && col < (int)matrix.size()) {
        if (x >= matrix[row][col]) {
            cnt += row + 1;
            col++;
        } else {
            row--;
        }
    }
    return cnt;
}

int kthSmallest(vector<vector<int>>& matrix, int k) {
    int n = matrix.size();
    int lo = matrix[0][0];
    int hi = matrix[n-1][n-1];
    while (lo < hi) {
        int mid = lo + (hi - lo) / 2;
        if (count(matrix, mid) >= k) hi = mid;
        else lo = mid + 1;
    }
    return lo;
}
```

**Time Complexity:** O(n · log(max − min))
**Space Complexity:** O(1)

---

## Feedback Given

### Scoring

| Category | Score | Notes |
|---|---|---|
| Problem Understanding & Clarification | 3/5 | Asked duplicates mid-coding; should ask up front. |
| Approach & Thought Process | 4/5 | Default to max-heap (generic); strong recovery to binary-search-on-answer with one nudge. |
| Code Quality & Correctness | 4/5 | Clean and correct first try. Lost a point for O(n²) bounds scan. |
| Complexity Analysis | 4/5 | Correct O(n log(max−min)) / O(1). |
| Communication | 4/5 | Predicate articulation strong; convergence proof needed prompting. |

### What went well
- Strong binary-search-on-answer instinct after one nudge — consolidating from prior session.
- Code worked on first trace including duplicates.
- Clear predicate articulation.

### What to work on
- Lead with structural exploitation when problem advertises sorted rows/cols.
- Master staircase walk (O(n) count in sorted matrix) — high-yield primitive.
- Always carry a 1-line convergence proof when binary-searching on values.
- Sorted-structure bounds are free — don't spend O(n²) computing them.

**Time Taken: 41 minutes**
