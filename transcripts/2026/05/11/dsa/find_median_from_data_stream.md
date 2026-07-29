# DSA Round Transcript
**Date:** 2026-05-11
**Start Time:** 10:54
**End Time:** 11:09
**Duration:** 15 minutes
**Problem:** Find Median from Data Stream
**Topic:** Heap, Design
**Difficulty:** Hard

---

## Problem Statement

Design a data structure that supports:
- `addNum(int num)` — Add an integer from the data stream.
- `findMedian()` — Return the median of all elements so far.

The median is the middle value in an ordered list. If the size is even, the median is the average of the two middle values.

**Example:**
```
addNum(1)
addNum(2)
findMedian() → 1.5

addNum(3)
findMedian() → 2.0
```

**Constraints:**
- `addNum` and `findMedian` called up to `5 * 10^4` times
- `-10^5 <= num <= 10^5`

---

## Conversation Log

**Interviewer:** Note your start time and share it. [Presented problem]. Do you have any clarifying questions?

**Aayush:** Start time 10:54. A brute force approach is to push the element using insertion sort in a vector — O(n) addNum and findMedian becomes O(1).

**Interviewer:** Correct. What's the overall complexity if addNum is called n times? Can you do better?

**Aayush:** O(n²) total. We can use 2 heaps — one max, one min. Always push to maxHeap, when |maxHeap| - |minHeap| > 1 rebalance. Median is (maxH.top + minH.top)/2 if sizes equal else maxH.top().

**Interviewer:** Good structure. But there's a gap — you always push to maxH without checking if the number belongs there. [Showed counterexample with addNum(0) producing wrong median.]

**Aayush:** If incoming num > minH.top() push to minH and rebalance so maxH has one element more. If incoming num < maxH.top() push to maxH and rebalance.

**Interviewer:** Good invariant thinking. Handle two edge cases before coding: what if minH is empty, and what if num falls exactly between the two tops?

**Aayush:** [Coded the solution with "always push to left, move top to right, rebalance" pattern]

**Interviewer:** [Traced through examples — correct. Noted the elegant invariant.] What are time/space complexities and edge cases?

**Aayush:** addNum TC is O(log n), findMedian O(1), space O(n). Edge cases: 1) findMedian before addNum — should throw exception. 2) All numbers same. 3) Even numbers in stream.

**Interviewer:** Complexity correct. On #1 — your code crashes (UB from top() on empty heap), it doesn't throw a clean exception. Case #3 was already covered by the example. What about negative numbers?

**Aayush:** No issues with negative numbers.

**Interviewer:** Correct. End time?

**Aayush:** 11:09.

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <iostream>
#include <queue>
#include <vector>

using namespace std;

class MedianFinder {
private:
    priority_queue<int> leftHeap;
    priority_queue<int, vector<int>, greater<int>> rightHeap;

public:
    void addNum(int num) {
        leftHeap.push(num);
        rightHeap.push(leftHeap.top());
        leftHeap.pop();
        if (rightHeap.size() > leftHeap.size()) {
            leftHeap.push(rightHeap.top());
            rightHeap.pop();
        }
    }

    double findMedian() {
        if (leftHeap.size() > rightHeap.size()) {
            return leftHeap.top();
        }
        return (leftHeap.top() + rightHeap.top()) / 2.0;
    }
};

int main() {
    MedianFinder mf;
    mf.addNum(1);
    mf.addNum(2);
    cout << "Median: " << mf.findMedian() << endl;
    mf.addNum(3);
    cout << "Median: " << mf.findMedian() << endl;
    return 0;
}
```

**Optimal Solution (if different):** Aayush's solution matches the optimal approach. Missing only an empty-stream guard in `findMedian()`.

**Time Complexity:** O(log n) addNum, O(1) findMedian
**Space Complexity:** O(n)

---

## Feedback Given

### 1. Problem Understanding & Clarification — 2/5
No clarifying questions asked — jumped straight to brute force. For a design problem should ask: what happens on empty stream, memory constraints, expected call volume. Empty stream turned out to be a real bug in the code.

### 2. Approach & Thought Process — 4.5/5
Correct brute force with accurate complexity, then immediately identified two heaps when prompted. Needed one correction on insertion logic but arrived at a cleaner implementation — "always push to left, always move top to right, rebalance" avoids explicit cross-heap comparisons entirely.

### 3. Code Quality & Correctness — 4/5
Clean, idiomatic C++ with correct class structure. Only bug: `findMedian()` crashes on empty stream (UB from calling `top()` on empty heap) with no guard.

### 4. Complexity Analysis — 5/5
Correct and immediate. O(log n) addNum, O(1) findMedian, O(n) space.

### 5. Communication — 3/5
Three edge cases volunteered (better than last round). But even-count case was already covered by the example. Empty stream was a good catch. Negative numbers handled correctly when probed.

### Overall: 18.5 / 25

**Strengths:** Fast recognition of two-heap pattern, elegant final implementation, flawless complexity analysis.

**To improve:** Ask clarifying questions on design problems especially around empty/null states — found the empty stream bug only when prompted, not proactively.

**Time Taken: 15 minutes**
