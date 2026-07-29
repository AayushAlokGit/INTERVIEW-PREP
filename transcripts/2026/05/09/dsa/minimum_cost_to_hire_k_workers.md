# DSA Round Transcript
**Date:** 2026-05-09
**Start Time:** 10:16
**End Time:** 10:55
**Duration:** 39 minutes
**Problem:** Minimum Cost to Hire K Workers
**Topic:** Greedy + Heap
**Difficulty:** Hard

---

## Problem Statement

There are `n` workers. You are given two arrays:
- `quality[i]` — the quality of worker `i`
- `wage[i]` — the minimum wage expectation of worker `i`

You want to hire exactly `k` workers to form a paid group under these rules:
1. Every worker in the group must be paid at least their minimum wage expectation.
2. The pay of each worker must be proportional to their quality.

Return the minimum total cost to hire exactly `k` workers. Answers within `10^-5` are accepted.

**Constraints:**
```
1 <= k <= n <= 10^4
1 <= quality[i] <= 10^4
1 <= wage[i] <= 10^6
```

**Example 1:**
```
quality = [10, 20, 5], wage = [70, 50, 30], k = 2
Output: 105.0
```

**Example 2:**
```
quality = [3, 1, 10, 10, 1], wage = [4, 8, 2, 2, 7], k = 3
Output: 30.66667
```

---

## Conversation Log

**Interviewer:** Note the current time, then here's your problem: Minimum Cost to Hire K Workers. [Full problem statement above.]

**Aayush:** 10:16

**Interviewer:** Start Time: 10:16. Any clarifying questions or ready to think through an approach?

**Aayush:** What are the constraints?

**Interviewer:** [Shared constraints.] Anything else?

**Aayush:** Can you give a hint?

**Interviewer:** If you fix which worker sets the wage-to-quality ratio for the group, what does the total cost simplify to?

**Aayush:** Why wage to quality ratio?

**Interviewer:** The proportionality rule means everyone is paid r × quality[i]. For this to satisfy minimum wages, r >= wage[i]/quality[i] for all workers in the group. To minimize cost, r equals the max ratio in the group — set by one worker. So total cost = ratio × (sum of qualities).

**Aayush:** This problem is going completely out of my head.

**Interviewer:** Sort workers by ratio ascending. Worker i (current) sets the ratio. You want k-1 smallest qualities from workers 0..i-1 plus worker i. What data structure maintains the k smallest qualities efficiently?

**Aayush:** Max heap.

**Interviewer:** Exactly. Maintain a max-heap of size k. For each worker i (sorted by ratio): push quality[i], if heap > k pop the largest, once heap == k compute cost = ratio[i] × sum. Go ahead and code it.

**Aayush:** [Submitted correct solution with pre-fill loop for first k-1 workers and main loop from k-1 to n-1.]

**Interviewer:** Traced Example 2: output 30.67 ✓. Logic is sound. What's the time and space complexity?

**Aayush:** TC is O(n log n for sorting + n log k in loop). SC is O(k).

**Interviewer:** Correct. Edge cases — give me at least three.

**Aayush:** Can't think of any right now.

**Interviewer:** [Walked through k=n, all same quality, all same ratio, k=1.]

**Aayush:** 10:55

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> quality {3, 1, 10, 10, 1};
    vector<int> wage{4, 8, 2, 2, 7};
    int k = 3;
    int n = quality.size();
    priority_queue<int> maxH;
    
    vector<vector<int>> workers(n);
    for(int i = 0; i < n; i++) {
        workers[i] = {wage[i], quality[i]};
    }
    
    sort(workers.begin(), workers.end(), [](const vector<int>& w1, const vector<int>& w2) {
        return (w1[0] / ((double)w1[1])) < (w2[0] / ((double)w2[1]));
    });
    
    double ans = INT_MAX;
    int sum = 0;
    
    for(int i = 0; i < k-1; i++) {
        maxH.push(workers[i][1]);
        sum += workers[i][1];
    }
    
    for(int i = k-1; i < n; i++) {
        while(!maxH.empty() && maxH.size() > k-1) {
            sum -= maxH.top();
            maxH.pop();
        }
        
        double ratio = workers[i][0] / ((double)workers[i][1]);
        sum += workers[i][1];
        maxH.push(workers[i][1]);
        ans = min(ans, ratio * sum);
    }
    
    cout << ans << endl;
    return 0;
}
```

**Optimal Solution (if different):** Same approach — Aayush reached the correct solution (with hints).

**Time Complexity:** O(n log n + n log k)
**Space Complexity:** O(k)

---

## Feedback Given

### Problem Understanding & Clarification — 3/5
Asked for constraints. No further clarifying questions — floating point tolerance and value distinctness worth flagging.

### Approach & Thought Process — 2/5
Could not derive the approach independently. Needed scaffolding for: the ratio insight, sorting by ratio, and the max-heap suggestion. Once explained, executed correctly. The greedy + heap archetype (sort by key parameter, maintain best-k subset) needs to be added to pattern library.

### Code Quality & Correctness — 4.5/5
Clean, correct code with no bugs. Pre-fill + main loop structure was well designed.

### Complexity Analysis — 5/5
Spot on: O(n log n + n log k) time, O(k) space. No hesitation.

### Communication — 2.5/5
Could not volunteer any edge cases. Required full prompting. Needs a deliberate edge case checklist habit: boundary k values (k=1, k=n), all-equal inputs, degenerate distributions.

### Overall: 3.4 / 5

**Key takeaways:**
- Greedy + heap pattern: sort by ratio, maintain best-k with a max-heap. Add to pattern library.
- Edge cases must be a deliberate habit, not an afterthought.

**Time Taken: 39 minutes**
