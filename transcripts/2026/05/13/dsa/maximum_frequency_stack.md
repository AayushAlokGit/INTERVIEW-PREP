# DSA Round Transcript
**Date:** 2026-05-13
**Start Time:** 19:15
**End Time:** 20:04
**Duration:** 49 minutes
**Problem:** Maximum Frequency Stack
**Topic:** Hash Map + Stack Design
**Difficulty:** Hard

---

## Problem Statement

Design a stack-like data structure that pushes and pops elements based on frequency.

Implement `FreqStack`:
- `FreqStack()` — initializes an empty stack
- `void push(int val)` — pushes val onto the stack
- `int pop()` — removes and returns the most frequently occurring element. If there is a tie, return the element closest to the top of the stack (most recently pushed).

**Constraints:**
- `0 <= val <= 10^9`
- At most `2 * 10^4` calls to push and pop
- It is guaranteed that pop is called on a non-empty stack

**Example:**
```
FreqStack fs;
fs.push(5); fs.push(7); fs.push(5); fs.push(7);
fs.push(4); fs.push(5);

fs.pop(); // returns 5 (freq 3, highest)
fs.pop(); // returns 7 (freq 2, tied with 5 — but 7 was pushed more recently at that freq)
fs.pop(); // returns 5 (freq 2)
fs.pop(); // returns 4 (freq 1, tied — but 4 was pushed more recently)
```

---

## Conversation Log

**Interviewer:** Note your start time and here's your next problem: [presented problem above]. Any clarifying questions before you start?

**Aayush:** 19:15

**Aayush:** int pop method — nothing needs to be done for the stack element which was popped, only need to return the most frequent element?

**Interviewer:** Yes — pop removes the element and returns it. The element's frequency also decreases by 1 after the pop.

**Aayush:** Does pop return the most frequent element of the popped element?

**Interviewer:** pop removes and returns the most frequently occurring element from the stack. You choose which element to remove based on frequency (and recency as tiebreaker).

**Aayush:** We can keep pushing each element into a vector and for each element added maintain a mapping of element to indices. For pop, iterate through the elements mapping and push (frequency, lastOccurringIndex, element) into a maxHeap. Pop the top to get the desired tuple, then remove the last index from the mapping.

**Interviewer:** Correct in terms of output. What's the time complexity of your pop operation?

**Aayush:** pop operation time complexity is O(n log n) to construct the heap.

**Interviewer:** Right — O(d log d) per pop, rebuilt every time. Can you make both push and pop O(1)? Hint: instead of computing frequency at pop-time, what if you maintained data structures that let you look up the answer instantly?

**Aayush:** Maintain 2 maps — freqMap (element → frequency) and freqToElements (freq → vector of elements). When pushing: freqMap[ele]++, freqToElements[freqMap[ele]].push_back(ele). When popping: get rbegin() of freqToElements (highest freq), pop last element, remove last element from vector, decrement maxFreq if vector empty.

**Interviewer:** Very close. Two things: (1) After popping, do you update freqMap for the element? (2) You're using rbegin() on std::map — what's the actual complexity?

**Aayush:** It is O(log N) to rearrange the red-black trees used for std::maps.

**Interviewer:** Exactly O(log n). To get true O(1), avoid the sorted map entirely. What single extra variable would let you instantly know the highest frequency?

**Aayush:** Maintain a maxFreq variable.

**Interviewer:** Exactly. Swap map for unordered_map and use maxFreq directly. Code it up.

**Aayush:** [asked for scaffold code]

**Interviewer:** No — write from scratch. You have all the pieces: unordered_map<int,int> freqMap, unordered_map<int,vector<int>> freqToElements, int maxFreq.

**Aayush:** [submitted first version — auto vec copy bug, missing freqMap[ele]--, missing erase of stale entry, still using std::map]

**Interviewer:** Two issues: (1) auto vec is a copy — use a reference. (2) Missing freqMap[ele]-- and stale entry not erased. Also still using std::map.

**Aayush:** [submitted second version — added freqMap[ele]--, added erase, still copy, still map]

**Interviewer:** Still a copy. Also demonstrated why missing freqMap[ele]-- matters with a push/pop/push trace. Pointed out stale entry bug.

**Aayush:** fix done [submitted third version — functionally correct but still std::map, still copy (though logic compensates)]

**Interviewer:** Logic is now correct. Still using std::map — switch to unordered_map. What's the final time and space complexity?

**Aayush:** O(1) time for push and pop. Space is O(n) where n is number of distinct elements.

**Interviewer:** Time correct once using unordered_map. Space — push 5 three times, it appears in freq buckets 1, 2, and 3. So space is proportional to what?

**Aayush:** Total pushes.

**Interviewer:** Correct. Note end time.

**Aayush:** 20:04

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

class FreqStack{
    int maxFreq;
    map<int, int> freqMap;
    map<int,vector<int>> freqToElementsIndex;

  public:

      FreqStack()
      {
          this->maxFreq = 0;
      }

      void push(int val)
      {
          freqMap[val]++;
          int f = freqMap[val];
          maxFreq = max(maxFreq, f);
          freqToElementsIndex[f].push_back(val);
      }

      int pop()
      {
         auto vec = freqToElementsIndex[maxFreq];
         int ele = vec.back();
         vec.pop_back();
         freqMap[ele]--;

         if(vec.size() == 0)
         {
             freqToElementsIndex.erase(maxFreq);
             maxFreq--;
         }
         else
         {
            freqToElementsIndex[maxFreq] = vec;
         }
         return ele;
      }
};
```

**Optimal Solution:**
```cpp
class FreqStack {
    int maxFreq = 0;
    unordered_map<int,int> freqMap;
    unordered_map<int,vector<int>> freqToElements;

public:
    void push(int val) {
        freqMap[val]++;
        int f = freqMap[val];
        maxFreq = max(maxFreq, f);
        freqToElements[f].push_back(val);
    }

    int pop() {
        auto& vec = freqToElements[maxFreq];
        int ele = vec.back();
        vec.pop_back();
        freqMap[ele]--;
        if(vec.empty()) {
            freqToElements.erase(maxFreq);
            maxFreq--;
        }
        return ele;
    }
};
```

**Time Complexity:** O(1) push and pop (with unordered_map)
**Space Complexity:** O(n) where n is total pushes

---

## Feedback Given

### Problem: Maximum Frequency Stack
**Topic:** Hash Map + Stack Design | **Difficulty:** Hard

### 1. Problem Understanding & Clarification — 3/5
Improvement — asked two clarifying questions before diving in. Still missed: negative values for val, confirming frequency decrements on pop.

### 2. Approach & Thought Process — 3/5
Started at O(d log d) — correct but suboptimal. Identified sorted map improvement without prompting. Needed two hints to reach O(1): maxFreq variable and unordered_map. Core insight arrived at, but not independently.

### 3. Code Quality & Correctness — 2/5
Three iterations to reach correct code. auto vec copy bug, missing freqMap[ele]--, missing stale erase — all discovered externally. Never switched to unordered_map despite being told three times.

### 4. Complexity Analysis — 4/5
Time correct. Space needed one probe — initially said distinct elements, corrected to total pushes.

### 5. Communication — 3/5
Clarifying questions improving. Edge case coverage still absent — never traced the given example before declaring done.

### Overall: 15/25

**Strengths:** Clarifying questions improving; complexity mostly solid; got to correct solution.
**Focus areas:** Stop using copies when you need references. Trace the given example once before saying done. When told to switch a data structure, do it.
