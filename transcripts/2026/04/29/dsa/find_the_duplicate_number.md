# DSA Round Transcript
**Date:** 2026-04-29
**Start Time:** 11:25 AM
**End Time:** 12:02 PM
**Duration:** 37 minutes
**Problem:** Find the Duplicate Number
**Topic:** Floyd's Cycle Detection
**Difficulty:** Hard

---

## Problem Statement

Given an array `nums` of `n + 1` integers where each integer is in the range `[1, n]` inclusive, there is **exactly one duplicate number**. Return that duplicate.

**Constraints:**
- You **must not modify** the array
- You must use only **O(1) extra space**
- Your solution must run in **O(n) time**

**Example 1:**
```
Input:  nums = [1, 3, 4, 2, 2]
Output: 2
```

**Example 2:**
```
Input:  nums = [3, 1, 3, 4, 2]
Output: 3
```

---

## Conversation Log

**Interviewer:** Please note the current time first. Presented the Find the Duplicate Number problem with O(1) space, O(n) time, no-modification constraints.

**Aayush:** 11:25 AM

**Interviewer:** Start time recorded. Asked for initial approach.

**Aayush:** A simple approach is storing frequency counts of array elements in O(n), then iterate through count array to get the duplicate.

**Interviewer:** That violates O(1) space constraint. Hinted at thinking of nums[i] as a pointer.

**Aayush:** Iterate through the array, mark the element at nums[abs(nums[i])] as negative. At the end the duplicate element's bit will be positive. If element at index j is positive then j is the duplicate.

**Interviewer:** Smart — but violates "must not modify the array" constraint. Reiterated the pointer hint.

**Aayush:** Another approach: have n bits (all 0), at index i perform XOR of nums[i]th bit with 1. At the end all bits will be 1 except one which would be 0. The position of this 0 bit gives the duplicate.

**Interviewer:** Two issues: (1) O(n) space for the bit array, (2) correctness breaks when duplicate appears even number of times — multiple bits would be 0. Gave Floyd's cycle detection hint explicitly.

**Aayush:** The problem guaranteed that the duplicate would appear only once since there are n+1 elements with 1 duplicate.

**Interviewer:** Corrected this reasoning — n+1 elements with values in [1,n] guarantees at least one duplicate but NOT that it appears exactly twice. Counterexample: nums = [1,1,1,1,2] satisfies all constraints but 1 appears 4 times. Gave Floyd's algorithm hint explicitly.

**Aayush:** Iterate through array to construct linked list. Head at index 0. Start slow and fast from head, move slow by one and fast by two. while(fast->next && fast->next->next). If slow == fast break, cycle found.

**Interviewer:** Correct phase 1. Clarified: no need to construct linked list (array IS the linked list implicitly). Pointed out phase 2 is needed to find cycle entry (the duplicate) — reset one pointer to 0 and advance both by one step until they meet.

**Aayush:** (Submitted code with both phases implemented correctly)

**Interviewer:** Code is correct. Traced through:
```
nums = [1,3,4,2,2]
Phase 1: slow=0,fast=0 → slow=1,fast=3 → slow=3,fast=4 → slow=2,fast=4 → slow=4,fast=4 (meet)
Phase 2: slow=0,fast=4 → slow=1,fast=2 → slow=3,fast=4 → slow=2,fast=2 (meet at duplicate=2) ✓
```
Noted code smell: while condition always true since values never equal n+1. Asked why phase 2 works mathematically.

**Aayush:** Not sure of the math behind it.

**Interviewer:** Explained:
- F = steps from start to cycle entry
- a = steps from cycle entry to meeting point
- C = cycle length
- Phase 1: 2(F+a) = F+a+kC → F = kC - a
- Phase 2: slow travels F steps; fast travels F = kC-a more steps from a into cycle → lands at kC from entry = 0 (cycle entry)
- They meet at the cycle entry = the duplicate

**Aayush:** 12:02 PM (end time)

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> nums{1,3,4,2,2};
    int n = nums.size()-1;
    int slow = 0, fast = 0;
    bool cycle = false;
    // i -> next ~ nums[i]
    while(nums[fast] != n+1 && nums[nums[fast]] != n+1)
    {
        slow = nums[slow];
        fast = nums[nums[fast]];
        if(slow == fast) 
        {
            cycle = true;
            break;
        }
    }
    slow = 0;
    if(cycle)
    {
        while(slow!=fast)
        {
            slow = nums[slow];
            fast = nums[fast];
        }
        cout<<slow;
    }
    return 0;
}
```

**Optimal Solution (cleaner version):**
```cpp
int findDuplicate(vector<int>& nums) {
    int slow = 0, fast = 0;
    // Phase 1: find meeting point inside cycle
    do {
        slow = nums[slow];
        fast = nums[nums[fast]];
    } while (slow != fast);
    // Phase 2: find cycle entry (the duplicate)
    slow = 0;
    while (slow != fast) {
        slow = nums[slow];
        fast = nums[fast];
    }
    return slow;
}
```

**Time Complexity:** O(n)
**Space Complexity:** O(1)

---

## Feedback Given

**Problem:** Find the Duplicate Number | **Difficulty:** Hard | **Topic:** Floyd's Cycle Detection

### Problem Understanding & Clarification — 3.5/5
Understood the core problem well. However, initially missed that the duplicate could appear more than twice (incorrectly assumed n+1 elements implies exactly one extra copy). This affected the correctness analysis of the bit-flip approach.

### Approach & Thought Process — 4/5
Good progression through approaches — frequency array → negate in-place → bit-flip — each ruled out for the right reasons. Needed the Floyd's hint to get to the optimal solution, but implemented it cleanly once pointed in the right direction.

### Code Quality & Correctness — 4/5
Floyd's implementation was correct and produced the right output. Clean two-phase structure. One code quality issue: the while condition `nums[fast] != n+1` is always true and misleading — the real termination is the `break` inside. Should be `while(true)` or restructured with a do-while.

### Complexity Analysis — 4/5
Correctly identified O(n) time. Implicitly O(1) space (only slow, fast, cycle variables). Could have stated space complexity explicitly.

### Communication — 3.5/5
Proposed approaches clearly but didn't explain the why behind phase 2 of Floyd's. In an interview, always explain the intuition — especially for non-obvious algorithms.

### Overall: 3.75 / 5

**Strengths:** Clean implementation, good elimination of naive approaches, quick to code once direction was clear.

**To work on:** Verify problem constraints carefully before committing to an approach. Know Floyd's phase 2 math — it's a common follow-up.

**Time Taken: 37 minutes**
