# DSA Round Transcript
**Date:** 2026-04-29
**Start Time:** 9:29 AM
**End Time:** 11:06 AM
**Duration:** 97 minutes
**Problem:** Task Scheduler
**Topic:** Greedy / Heap Simulation
**Difficulty:** Medium-Hard

---

## Problem Statement

You are given a list of CPU tasks, each labeled with a letter `A` to `Z`, and a non-negative integer `n` representing the **cooldown period** between two same tasks.

In each interval, the CPU can either:
- Execute a task, or
- Stay **idle**

Tasks of the **same type** must be separated by at least `n` intervals.

Return the **minimum number of intervals** the CPU will take to finish all tasks.

**Example 1:**
```
Input:  tasks = ["A","A","A","B","B","B"], n = 2
Output: 8
Explanation: A -> B -> idle -> A -> B -> idle -> A -> B
             (8 intervals total)
```

**Example 2:**
```
Input:  tasks = ["A","A","A","B","B","B"], n = 0
Output: 6
Explanation: No cooldown needed, just run all 6 tasks back to back.
```

**Example 3:**
```
Input:  tasks = ["A","A","A","A","B","B","C","C"], n = 2
Output: 10
```

**Constraints:**
- `1 <= tasks.length <= 10^4`
- `tasks[i]` is an uppercase English letter
- `0 <= n <= 100`

---

## Conversation Log

**Interviewer:** Hey Aayush! Welcome to your DSA round. Please note the current time and share it with me — I'll use that as the official start time. Then I presented the Task Scheduler problem.

**Aayush:** I have solved Minimum Window Substring (first problem presented), give me another non-traditional problem.

**Interviewer:** Presented Task Scheduler problem. Asked for start time.

**Aayush:** 9:29 AM

**Interviewer:** Start time recorded. Asked for initial approach.

**Aayush:** We want to finish off more frequent tasks as early as possible otherwise if they are done at the end the wait time would increase.

**Interviewer:** Good instinct. Asked to build on it — how to structure the schedule around the most frequent task, and what determines the minimum total intervals.

**Aayush:** I was thinking that at each slot of the schedule we pick up the most frequent task. If this can be scheduled (decided if the next viable time < current time) then schedule it, else pick up the next most frequent task and try to schedule it. We keep on getting the next most frequent task until we find a task that can be scheduled. Now if a task can be scheduled then it is next viable to be scheduled at current_time + n. And if no task can be scheduled then the current slot must be idle.

**Interviewer:** Solid approach — that's essentially a max-heap + cooldown queue simulation. Asked about data structures: what to use for most-frequent lookup, and how to track cooldowns.

**Aayush:** We could have a maxHeap of pair<int,char> where the first part would be the frequency of the task. And we can store the next viable time for each task in a map<char,int>. During each iteration of trying to schedule in a slot we would keep popping from maxHeap, if the current task can be scheduled go ahead and schedule it, else get the next task which can be scheduled.

**Interviewer:** Good — data structures are clear. Warned about pushing back temporarily popped tasks. Asked to code it up.

**Aayush:** (Submitted first version of code with two bugs: off-by-one in cooldown `currTime + n` instead of `currTime + n + 1`, and missing semicolon after do-while)

**Interviewer:** Good structure overall. Spotted one critical bug and one syntax error. Gave hints — asked to trace through example manually to find the cooldown bug, and to look closely at the do-while syntax.

**Aayush:** (Submitted fixed version with `currTime + n + 1` and semicolon added, also changed char to string)

**Interviewer:** Traced through the fixed code:
```
t=0: B scheduled  → nextViable[B] = 3
t=1: A scheduled  → nextViable[A] = 4
t=2: idle         (both A and B blocked)
t=3: B scheduled  → nextViable[B] = 6
t=4: A scheduled  → nextViable[A] = 7
t=5: idle
t=6: B scheduled
t=7: A scheduled
Output: [B, A, _, B, A, _, B, A] → size = 8 ✓
```
Code is correct. Asked about time and space complexity and whether O(N) solution exists.

**Aayush:** Time complexity is O(n log n) and space complexity is O(n).

**Interviewer:** Pushed back — heap has at most 26 elements so operations are O(log 26) = O(1). Real time complexity is O(result) = O(N × n) in worst case. Space is O(1) for heap/maps, O(result) for schedule vector. Asked about O(N) mathematical formula.

**Aayush:** There would at least be maxFreq*(n+1) slots for the scheduling task with max frequency and the lesser frequency tasks would be scheduled in between the slots.

**Interviewer:** Close! Pointed out: last frame doesn't need trailing idle, multiple tasks can have max frequency (count_max), and dense task sets may need no idles at all. Asked to refine the formula.

**Aayush:** Not able to get it.

**Interviewer:** Revealed the formula:
```
result = max(tasks.size(), (f_max - 1) * (n + 1) + count_max)
```
With explanation of frame-based thinking and the dense-tasks edge case.

**Aayush:** 11:06 AM (end time)

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<string> tasks({"A","A","A","B","B","B"});
    int n = 2;
    map<string,int> nextViableTime, freq;
    priority_queue<pair<int,string>> maxH;
    
    int currTime = 0;
    int taskCount = tasks.size();
    vector<string> schedule;
    for(string ch:tasks)
    {
        freq[ch]++;
        nextViableTime[ch] = 0;
    }
    
    for(auto [ch,frequency]: freq)
    {
        maxH.push({frequency,ch});
    }
    
    while(taskCount > 0)
    {
        string taskToBeScheduled = "_";
        vector<pair<int,string>> tmp;
        do
        {
            auto [taskFreq, taskChar] = maxH.top();
            maxH.pop();
            if(nextViableTime[taskChar] <= currTime)
            {
                taskToBeScheduled = taskChar;
                taskFreq--;
                taskCount--;
                nextViableTime[taskToBeScheduled] = currTime + n + 1;
                if(taskFreq > 0)
                {
                    maxH.push({taskFreq, taskChar});
                }
                break;
            }
            else
            {
                tmp.push_back({taskFreq, taskChar});    
            }
        }while(maxH.size() > 0);
        
        currTime++;
        schedule.push_back(taskToBeScheduled);
        if(tmp.size() > 0)
        {
            for(auto [taskFreq, taskChar]:tmp) maxH.push({taskFreq,taskChar});
        }
        tmp.clear();
    }
    for(string ch:schedule) cout<<ch<<" ";
    cout<<endl;
    return 0;
}
```

**Optimal Solution:**
```cpp
int leastInterval(vector<char>& tasks, int n) {
    int freq[26] = {};
    for (char t : tasks) freq[t - 'A']++;
    int f_max = *max_element(freq, freq + 26);
    int count_max = count(freq, freq + 26, f_max);
    return max((int)tasks.size(), (f_max - 1) * (n + 1) + count_max);
}
```

**Time Complexity:** O(n log n) (Aayush's answer) — Corrected: O(result) = O(N × cooldown) since heap is bounded by 26 → O(1) per operation
**Space Complexity:** O(n) (Aayush's answer) — Corrected: O(1) for heap/maps (bounded by 26), O(result) for schedule vector

---

## Feedback Given

**Problem:** Task Scheduler | **Difficulty:** Medium-Hard | **Topic:** Greedy / Heap Simulation

### Problem Understanding & Clarification — 4/5
Quickly grasped the core constraint (cooldown between same tasks) and didn't need much hand-holding on the problem setup. Could have asked one clarifying question upfront (e.g., "are tasks case-sensitive?" or "can n be 0?") to show thoroughness.

### Approach & Thought Process — 4/5
Strong greedy intuition — "schedule most frequent task first to avoid pushing costly tasks to the end" is exactly right. Arrived at the heap + cooldown map structure independently and cleanly. Missed the mathematical O(N) insight on own, but that's expected — it requires a specific "aha" moment.

### Code Quality & Correctness — 3.5/5
The simulation logic was correct in structure. Caught and fixed both bugs (off-by-one in cooldown, missing semicolon) quickly when hinted. Minor issues: `return 0` instead of `return schedule.size()` in main, and using `string` instead of `char` unnecessarily added overhead.

### Complexity Analysis — 3/5
Initial answer of O(n log n) wasn't precise. The key insight — heap is bounded by 26 characters so operations are O(1) — was missed. The actual complexity O(result) = O(N × n) in worst case is non-obvious but important to articulate.

### Communication — 4/5
Thought aloud well and explained approach clearly before coding. Good habit. Could be slightly more structured (e.g., state approach → data structures → walkthrough → code).

### Overall: 3.7 / 5

**Strengths:** Greedy intuition, clean code structure, quick bug fixing.

**To work on:** Complexity analysis precision, pushing to O(1) space / formula-based optimization before being prompted.

**Time Taken: 97 minutes**
