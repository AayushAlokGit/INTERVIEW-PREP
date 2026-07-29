# DSA Round Transcript
**Date:** 2026-07-14
**Start Time:** 12:42
**End Time:** 13:58
**Duration:** 76 minutes
**Problem:** Number of Submatrices That Sum to Target (LeetCode 1074)
**Topic:** Prefix Sums + Hashmap (2D → 1D reduction)
**Difficulty:** Hard

---

## Problem Statement

Given a 2D matrix of integers `matrix` and an integer `target`, return the number of non-empty submatrices that sum to `target`.

A submatrix `(x1, y1, x2, y2)` is the set of all cells `matrix[x][y]` with `x1 <= x <= x2` and `y1 <= y <= y2`.

Two submatrices `(x1, y1, x2, y2)` and `(x1', y1', x2', y2')` are different if they have some coordinate that is different — e.g. if `x1 != x1'`, they are different submatrices regardless of whether the values inside are the same.

**Example 1:**
```
Input:  matrix = [[0,1,0],
                  [1,1,1],
                  [0,1,0]],
        target = 0
Output: 4
```
The four 1x1 submatrices that contain a `0` each sum to 0.

**Example 2:**
```
Input:  matrix = [[1,-1],
                  [-1,1]],
        target = 0
Output: 5
```
The two 1x2 submatrices, the two 2x1 submatrices, and the one 2x2 submatrix all sum to 0.

**Example 3:**
```
Input:  matrix = [[904]], target = 0
Output: 0
```

**Constraints given on request:**
- `1 <= rows, cols <= 100`
- `-1000 <= matrix[i][j] <= 1000` (values can be negative)
- `-10^8 <= target <= 10^8`

---

## Conversation Log

**Interviewer:** Please note the current time and share it with me. Then: here's the problem. Any clarifying questions before you start thinking about an approach?

**Aayush:** 12:42

**Interviewer:** Start time recorded. Any clarifying questions about the problem before you dive into an approach?

**Aayush:** what are the constraints?

**Interviewer:** `1 <= rows, cols <= 100`; `-1000 <= matrix[i][j] <= 1000` — values can be negative; `-10^8 <= target <= 10^8`. Anything else you want to pin down?

**Aayush:** we can precompute a matrix s where s[i][j] = sum of elements (x,y) where 0<=x<=i and 0<=y<=j. Now we can fix x1 and y1 and then choose x1<=x2<=rows and y1<=y2<cols. Then the sum of the elements of the submatrix (x1,x2,y1,y2) would be s[x2][y2] + s[x1][y1] - s[x2][y1] - s[x1][y2] and if this equals target we increment count. The time complexity of this brute force is O(n^4)

**Interviewer:** Good — the 2D prefix-sum + fix-both-corners idea is right, and O(n^4) is the correct count (≈10^8 at n=100, borderline but not absurd). Two things:

1. **Verify your formula.** Take the 3×3 matrix from Example 1 and plug in `x1=1, y1=1, x2=1, y2=1` (the single center cell, value 1). What does your formula produce, and what should it produce?
2. Once you've settled that — can you do better than O(n^4)? Think about what you're recomputing.

**Aayush:** s[x2][y2] + s[x1-1][y1-1] - s[x1-1][y2] - s[x2][y1-1], x1,y1 >= 1. If x1 == 0, required sum = s[0][y2] - s[0][y1-1], y1 >= 1

**Interviewer:** The general formula is right now. But look at your edge case: if `x1 = 0` and `x2 = 2`, your expression only reads row `0` of the prefix matrix. Is that the sum of rows 0 through 2, or just row 0? Also — rather than special-casing `x1==0` and `y1==0` separately (and the both-zero case), is there a cleaner way to make the general formula always valid?

**Aayush:** pad the prefix matrix with one row and one column of zeros — this will avoid all edge case issues with x1 and x2

**Interviewer:** Exactly. Now the real question: O(n^4) is 10^8 ops. Can you do better? Hint: you're iterating over four independent indices. Try fixing just *two* of the four and asking — given those two are fixed, what problem am I actually solving over the remaining two?

**Aayush:** we can fix x1 and x2 then y can vary from 0 to cols. When y moves by one all the elements in one column between x1 and x2 are added to the matrix sum. So if at a particular y we find that curSum - target has already been found then we can increment the counter and then push the currentSum to a map. This will ensure we get all the counts.

**Interviewer:** That's the key insight — fix the row pair, collapse the strip into a 1D array, and it becomes subarray-sum-equals-target with a hashmap. Two details before you code:
1. You said "if `curSum - target` has already been found, increment the counter." Increment it by how much? Consider `[1, -1, 1, -1, 1]` with target `1`.
2. Does the map get created once, or does something have to happen to it as `(x1, x2)` changes?

And give me the time and space complexity.

**Aayush:** increment map[cur-target] and reset the map for each x1 and x2 pair

**Interviewer:** Right on both. You skipped my third question though — what's the time and space complexity? And one more: what does the map need to contain *before* you start scanning the columns for a given row pair, so that a submatrix starting at column 0 gets counted?

**Aayush:** 0 sum with count of 1

**Interviewer:** Correct — seed it with `{0: 1}`. You've now dropped the complexity question twice. Time and space complexity of your optimized approach? (rows = R, cols = C.)

**Aayush:** time complexity will be O(rows^2 * col) and we can have space of O(rows*cols) for the prefix sums matrix

**Interviewer:** Time is right. On space — is the prefix matrix the only auxiliary structure you allocate? What's the hashmap's worst-case size?

**Aayush:** hashmap is O(cols)

**Interviewer:** Right — O(C), dominated by the O(R·C) prefix matrix. Go ahead and write the full solution.

**Aayush:** *(submitted C++ solution — see Solution section)*

**Interviewer:** Code looks structurally sound. Before I comment, **dry-run it on Example 2** — `matrix = [[1,-1],[-1,1]]`, `target = 0`, expected 5. Show me the `pref` table, then for each `(x1,x2)` pair, the `curSum` values and what `ans` becomes. Don't reason about intent — execute your actual lines.

**Aayush:**
```
pref is {0,0,0} {0,1,-1} {0,0,0}
x1=1,x2=1 -> cur {0:2} {1:1} ans += 1
x1=1, x2=2 -> cur {0:3} {} ans += 2
x1=2, x2=2 -> cur {0:2} {-1:1}
```

**Interviewer:** Your `pref` is right. But add up your increments: 1 + 2 + (nothing stated for the last pair) — that doesn't reach 5. You're summarizing final map states instead of executing step by step. Do it properly for `x1=1, x2=2`: show me `curSum` and `ans +=` at `y=1`, then at `y=2`, with the map contents at each moment.

**Aayush:** it works

**Interviewer:** "It works" isn't a trace — and it's the thing I'd push back on hardest in a real interview. You told me `x1=1, x2=2` contributes `ans += 2`. I need you to show the two lines. For `x1=1, x2=2`, `target=0`, map starts `{0:1}`, `curSum=0`:
- `y=1`: `curSum += pref[2][1] - pref[0][1]` = ? → `curSum` = ? → `ans += cnt[curSum - 0]` = ? → map becomes ?
- `y=2`: same three steps.

**Aayush:**
```
y=1 curSum = 0, ans += cnt[0] = 1, cnt[0] = 2
y=2 curSum = 0, ans += cnt[0] = 2, cnt[0] = 3
```

**Interviewer:** There it is — that pair contributes **3**, not 2. So the total is 1 + 3 + 1 = **5** ✓. Your code is correct; your first trace wasn't. That's the takeaway: when you skipped the per-step execution and jumped to "it works," you reported the wrong number and didn't notice.

Three follow-ups on the code:
1. `ans += cnt[curSum - target];` uses `operator[]` on an `unordered_map`. What does that do when the key is absent? Is it a correctness problem?
2. Constraints: values in `[-1000, 1000]`, matrix up to 100×100. Is `int` safe for `curSum` and for `ans`?
3. Suppose the matrix were 1000 rows × 10 cols. Your loop is O(R²·C). Does your code have a preventable weakness there?

**Aayush:** operator inserts the key if not present but the value will be 0 so it should be safe. Yes it is safe since curSum can be at max rows*cols*10^3 ~ 10^7. We can iterate on x or y depending on which is greater — if cols is greater than rows iterate on x, else iterate on y.

**Interviewer:** Good on all three. `operator[]` inserts a default 0, so the read is safe — it does pollute the map with zero-valued keys (still O(C) bounded), but `cnt.count()` / `cnt.find()` is cleaner. `curSum` max ≈ 10^7 ✓, and `ans` is at most (100·101/2)² ≈ 2.55×10^7, so `int` holds both — worth stating both, not just `curSum`. And right: fix the pair over the **smaller** dimension, giving O(min²·max). For 1000×10 that's 10^5 instead of 10^7.

Please note the current time.

**Aayush:** 13:58

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<vector<int>> m{
        {-1,1,-1},
        {1,1,1},
        {1,1,1}
    };
    int target = 0;
    int rows = m.size();
    int cols = m[0].size();

    vector<vector<int>> pref(rows+1, vector<int>(cols+1, 0));

    for (int i = 0; i < rows; i++) {
        for (int j = 0; j < cols; j++) pref[i+1][j+1] = m[i][j];
    }

    // column-wise prefix: pref[i][j] = sum of column j over rows 1..i
    for (int i = 1; i <= rows; i++) {
        for (int j = 1; j <= cols; j++) {
            pref[i][j] += pref[i-1][j];
        }
    }

    int ans = 0;
    unordered_map<int,int> cnt;

    for (int x1 = 1; x1 <= rows; x1++) {
        for (int x2 = x1; x2 <= rows; x2++) {
            int curSum = 0;
            cnt[0] = 1;
            for (int y = 1; y <= cols; y++) {
                curSum += pref[x2][y] - pref[x1-1][y];
                ans += cnt[curSum - target];
                cnt[curSum]++;
            }
            cnt.clear();
        }
    }

    cout << "Ans is " << ans << endl;
    return 0;
}
```

**Correct.** Verified against Example 2 (`[[1,-1],[-1,1]]`, target 0 → 5).

**Optimal Solution (cleanups on top of his):**
```cpp
class Solution {
public:
    int numSubmatrixSumTarget(vector<vector<int>>& matrix, int target) {
        int R = matrix.size(), C = matrix[0].size();

        // Optional: transpose so R = min(R, C) => O(min^2 * max)
        vector<vector<int>> pref(R + 1, vector<int>(C + 1, 0));
        for (int i = 0; i < R; i++)
            for (int j = 0; j < C; j++)
                pref[i+1][j+1] = matrix[i][j] + pref[i][j+1];

        int ans = 0;
        for (int x1 = 1; x1 <= R; x1++) {
            for (int x2 = x1; x2 <= R; x2++) {
                unordered_map<int,int> cnt{{0, 1}};   // fresh per row pair
                int curSum = 0;
                for (int y = 1; y <= C; y++) {
                    curSum += pref[x2][y] - pref[x1-1][y];
                    auto it = cnt.find(curSum - target);  // no junk inserts
                    if (it != cnt.end()) ans += it->second;
                    cnt[curSum]++;
                }
            }
        }
        return ans;
    }
};
```

Key ideas:
- Pad the prefix table by one row/col of zeros so the general formula always applies — no `x1==0` / `y1==0` special cases.
- Fix the row pair `(x1, x2)`, collapse each column into a single value, and the 2D problem becomes **Subarray Sum Equals K** in 1D.
- Seed the map with `{0: 1}` and reset it per row pair; add `cnt[curSum - target]` (not `++`) since the same prefix sum can recur.
- Loop the squared dimension over the **smaller** side → O(min²·max).

**Time Complexity:** O(R² · C) — his answer, correct. (O(min² · max) with the transpose.)
**Space Complexity:** O(R · C) for the prefix matrix; hashmap is O(C), dominated — his answer, correct.

---

## Feedback Given

**Overall: solid technically, but slow, and the self-verification gap showed up again.**

### Problem Understanding & Clarification — 3.5/5
You asked for constraints, which is good. But you then **never used them** until I forced the overflow question at the end. Constraints exist to tell you what algorithm is admissible — "100×100, values up to 1000" immediately tells you O(n⁴)=10⁸ is borderline and that `int` needs a sanity check. You also didn't ask about negatives; I volunteered it, and it's the single most important fact in this problem (it's what kills any sliding-window idea and forces the hashmap).

### Approach & Thought Process — 4.5/5
Strongest part of the round. You got the 2D prefix sum brute force out fast, and when I nudged you toward "fix two of four indices," you found the row-pair-collapse-to-1D reduction **on your own** and correctly recognized it as subarray-sum-equals-k. You also nailed the `min²·max` asymmetry follow-up. This is a genuine improvement over your historical pattern of reaching for a generic template instead of the structure-exploiting one.

### Code Quality & Correctness — 4/5
The code is **correct**. Clean padding trick, correct per-pair map reset, correct `{0:1}` seed, correct `ans += cnt[curSum-target]` (not `++`). Two small things: `operator[]` on the read side inserts junk keys, and there was debug output left in. Also, your first written formula (`s[x2][y2] + s[x1][y1] - s[x2][y1] - s[x1][y2]`) was wrong — off by one on the exclusive corners — and you only fixed it because I made you plug in numbers.

### Complexity Analysis — 3.5/5
O(R²·C) time and O(R·C) space were both right, and you correctly identified the map as O(C). But **I had to ask three times.** You dropped the complexity sub-question twice while answering the other parts of the same message. In an interview that reads as either not listening or avoiding.

### Communication — 2.5/5
This is where the round hurt. When I asked you to dry-run Example 2, you gave me a summary of final map states, and your arithmetic didn't add up (you had `ans` reaching 3 or 4, not 5). When I pointed at the gap, you replied **"it works."** That is the worst possible answer. I had to ask a third time, forcing a blank-filling format, before you executed the code — and the moment you did, you immediately produced the correct `+3` and the correct total of 5.

Your code was right the whole time. But an interviewer who asks you to trace and gets "it works" will conclude you either can't trace or won't, and both are disqualifying. **The trace is not a formality — it's the deliverable.**

### Time
**Time Taken: 76 minutes** — roughly double what this should take. A LC-Hard you have the key insight for should land in 35-40 min. A large chunk of the overage was the back-and-forth on the trace.

### Top 2 things to fix
1. **Dry-run unprompted, and dry-run mechanically.** Pick the smallest example that exercises the tricky path, then execute *line by line* — not intent, not summary.
2. **Answer every sub-question asked.** When I ask two things, answer two things.
