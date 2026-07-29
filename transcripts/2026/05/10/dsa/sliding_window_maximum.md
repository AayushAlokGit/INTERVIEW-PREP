# DSA Round Transcript
**Date:** 2026-05-10
**Start Time:** 8:25 PM
**End Time:** 8:40 PM
**Duration:** 15 minutes
**Problem:** Sliding Window Maximum
**Topic:** Sliding Window, Monotonic Deque
**Difficulty:** Hard

---

## Problem Statement

You are given an array of integers `nums` and an integer `k`. There is a sliding window of size `k` moving from the left to the right of the array. At each position, you can only see the `k` numbers in the window. Return the max value in each window position.

**Example:**
```
Input:  nums = [1, 3, -1, -3, 5, 3, 6, 7], k = 3
Output: [3, 3, 5, 5, 6, 7]

Window positions:
[1  3  -1] -3  5  3  6  7  → max = 3
 1 [3  -1  -3] 5  3  6  7  → max = 3
 1  3 [-1  -3  5] 3  6  7  → max = 5
 1  3  -1 [-3  5  3] 6  7  → max = 5
 1  3  -1  -3 [5  3  6] 7  → max = 6
 1  3  -1  -3  5 [3  6  7] → max = 7
```

**Constraints:**
- `1 <= nums.length <= 10^5`
- `-10^4 <= nums[i] <= 10^4`
- `1 <= k <= nums.length`

---

## Conversation Log

**Interviewer:** Please note the current time and share it with me. Here's your problem: Sliding Window Maximum. [problem presented above]. Do you have any clarifying questions?

**Aayush:** Current time is 8:25 PM.

**Interviewer:** Noted — Start time: 8:25 PM. Any clarifying questions or ready to walk through your approach?

**Aayush:** The brute force approach is to compute max for every k window subarray in O(nk).

**Interviewer:** Correct — that's the brute force. O(nk) time, O(1) extra space. Can you think of a way to bring it down? What data structure might help you avoid recomputing the max from scratch for every window?

**Aayush:** A monotonically decreasing queue can be used. Here the front of the queue would be the index of maximum element of the window. At each index i we must ensure that queue has only indices in [i-k,i] so we remove from front those indices which don't fall in the range. Now while the element at queue.back() index is <= curr element we pop from the back to ensure the monotonically decreasing nature of the queue. Then push the element to the queue back.

**Interviewer:** Great approach — that's exactly the optimal idea. A few quick questions before you code: 1. You said remove from front indices that don't fall in range [i-k, i] — at what point in your loop do you do this — before or after appending the current element? 2. At what index i do you start appending to your result array?

**Aayush:** 1. Before. 2. index = k.

**Interviewer:** Both correct. Go ahead and code it up.

**Aayush:** [submitted C++ solution]

**Interviewer:** Good, the logic is correct. In the second loop, you check `dq.front() <= i-k` — should that be `<` or `<=`? Think about it: if `dq.front() == i-k`, is that index still inside the window `[i-k+1, i]`?

**Aayush:** It should be <= because i-k is not part of the window containing k elements ending at i -> [i-k+1, i].

**Interviewer:** Exactly right. Your code is correct. Now — time and space complexity?

**Aayush:** Time is O(n) processing each index at most once. And SC is O(k) for the deque.

**Interviewer:** Correct on both counts. One follow-up — your result array `ans` is O(n). Do you count that in your space complexity or not, and why?

**Aayush:** We could include that to make SC O(n + k).

**Interviewer:** Close — but since k <= n always, O(n + k) simplifies to just O(n). The k term is dominated. Typically: if you count output array → O(n) space. If output is excluded → O(k) for the deque only. Both are acceptable, just be explicit about which convention you're using. Please note the current time.

**Aayush:** 8:40 PM.

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> nums{1, 3, -1, -3, 5, 3, 6, 7};
    int k = 3;
    int n = nums.size();

    vector<int> ans;
    deque<int> dq;

    for (int i = 0; i < k; i++) {
        while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
        dq.push_back(i);
    }

    ans.push_back(nums[dq.front()]);

    for (int i = k; i < n; i++) {
        while (!dq.empty() && dq.front() <= i - k) dq.pop_front();
        while (!dq.empty() && nums[dq.back()] <= nums[i]) dq.pop_back();
        dq.push_back(i);
        ans.push_back(nums[dq.front()]);
    }

    for (int i : ans) cout << i << " ";
    return 0;
}
```

**Optimal Solution (if different):** Not applicable — Aayush's solution is optimal.

**Time Complexity:** O(n) — each index pushed and popped at most once
**Space Complexity:** O(k) for the deque; O(n) if output array is included

---

## Feedback Given

### 1. Problem Understanding & Clarification — 3/5
Jumped straight to brute force without asking any clarifying questions — no confirmation on whether duplicates are possible, whether the array can be empty, or what to return when k == n. This is a recurring pattern. Make it a habit to spend 60 seconds on clarifications before discussing approach.

### 2. Approach & Thought Process — 5/5
Excellent. Identified the monotonic deque approach immediately and described it precisely — front holds max index, prune from back to maintain decreasing order, prune from front to maintain window bounds. No hints needed.

### 3. Code Quality & Correctness — 4/5
Code was clean, well-structured, and correct. Good call splitting the first window initialization separately. One minor note — caught the <= vs < boundary correctly when challenged, but should be verifying these boundary conditions independently before declaring the solution done. Trace at least one edge case (e.g., k=1 or k=n) before submitting.

### 4. Complexity Analysis — 4/5
Correctly identified O(n) time and O(k) / O(n) space. Good that the output array was reasoned about separately. Remember: O(n+k) = O(n) — always simplify when one term dominates.

### 5. Communication — 4/5
Clear and concise throughout. Explained the deque invariant well upfront. Could volunteer edge cases more proactively.

### Overall: 20/25 — Strong performance
**Time Taken: 15 minutes**

Key takeaway: The one consistent gap is clarifying questions at the start and self-checking edge cases before declaring done. Both are easy wins in a real interview.
