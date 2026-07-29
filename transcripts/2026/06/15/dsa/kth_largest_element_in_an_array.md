# DSA Round Transcript
**Date:** 2026-06-15
**Start Time:** 15:49
**End Time:** 15:55
**Duration:** 6 minutes
**Problem:** Kth Largest Element in an Array
**Topic:** Heap / Quickselect
**Difficulty:** Medium

---

## Problem Statement
Given an integer array `nums` and an integer `k`, return the `k`th largest element in sorted order (not the kth distinct element).

**Example 1:** `nums = [3,2,1,5,6,4], k = 2` → `5`
**Example 2:** `nums = [3,2,3,1,2,4,5,5,6], k = 4` → `4`

**Constraints:**
```
1 <= k <= nums.length <= 10^5
-10^4 <= nums[i] <= 10^4
```

---

## Conversation Log

**Interviewer:** Presented the problem; asked for clarifying questions.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints; noted duplicates allowed, negatives possible, k always valid.

**Aayush:** Simple — sort and return the element at the kth position, O(n log n). Better — maintain a min-heap of size k, O(n log k) time, O(k) space.

**Interviewer:** Both correct. Can you do better than O(n log k) — an average-linear approach?

**Aayush:** Not sure how to bring it down to O(n).

**Interviewer:** Hint — quicksort's partition lands the pivot at its final sorted position; use that to home in without fully sorting.

**Aayush:** Not very familiar with quicksort.

**Interviewer:** Explained quickselect conceptually (partition, compare pivot index to k-1, recurse one side, average O(n), random pivot for worst case). Offered: attempt quickselect or code the min-heap.

**Aayush:** Min heap.

**Interviewer:** Code it.

**Aayush:** (submitted correct min-heap solution — push each, pop when size > k, print top)

**Interviewer:** Articulate the invariant — why is top() the kth largest at the end? Trace example 1, then give complexity.

**Aayush:** The min-heap ends with the k maximum elements (we always pop the min). The min among the k maximums is the answer. TC O(n log k), SC O(k).

**Interviewer:** Correct. Reinforce — one-line trace of example 1.

**Aayush:** Heap {5,6}, answer 5.

**Interviewer:** Correct. Solution complete; revealed quickselect reference for the O(n)-average optimum.

---

## Solution
**Aayush's Final Solution:**
```cpp
priority_queue<int, vector<int>, greater<int>> minHeap;
for (int num : nums) {
    minHeap.push(num);
    if (minHeap.size() > k) minHeap.pop();
}
cout << minHeap.top();   // kth largest
```

**Optimal Solution (revealed — quickselect, O(n) average):**
```cpp
int findKthLargest(vector<int>& nums, int k) {
    int target = nums.size() - k;
    int lo = 0, hi = nums.size() - 1;
    while (lo <= hi) {
        int p = partition(nums, lo, hi);
        if (p == target) return nums[p];
        else if (p < target) lo = p + 1;
        else hi = p - 1;
    }
    return -1;
}
int partition(vector<int>& nums, int lo, int hi) {
    int pivotIdx = lo + rand() % (hi - lo + 1);
    swap(nums[pivotIdx], nums[hi]);
    int pivot = nums[hi], i = lo;
    for (int j = lo; j < hi; j++)
        if (nums[j] < pivot) swap(nums[i++], nums[j]);
    swap(nums[i], nums[hi]);
    return i;
}
```

**Time Complexity:** O(n log k) (his heap solution)
**Space Complexity:** O(k)

---

## Feedback Given

### Strengths
- Immediate command of the solution space: stated sort (O(n log n)) and size-k min-heap (O(n log k), O(k)) with correct complexities unprompted.
- Clean, correct code on the first try — no bugs.
- Articulated the heap invariant precisely (heap holds k largest; min-heap top = smallest of those = kth largest).
- Precise complexity; self-traced the example cleanly when asked.

### Areas to work on
- Quickselect / partition-based selection is a knowledge gap (not a reasoning gap). High value — recurs in kth largest/smallest, median, top-k, k-closest. Drill Lomuto/Hoare partition + quickselect.

### Scoring (out of 5)
| Criterion | Score | Note |
|---|---|---|
| Problem understanding & clarification | 4.5 | Asked constraints; clocked duplicates/negatives |
| Approach & thought process | 4.0 | Two correct approaches instantly; couldn't reach quickselect (knowledge gap) |
| Code quality & correctness | 5.0 | Correct first try, no bugs |
| Complexity analysis | 5.0 | Precise O(n log k)/O(k), correct invariant |
| Communication | 4.5 | Articulated invariant clearly, self-traced |

**Time Taken: 6 minutes**
