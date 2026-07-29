# DSA Round Transcript
**Date:** 2026-05-19
**Start Time:** 10:01
**End Time:** 11:06
**Duration:** 65 minutes
**Problem:** Maximum Number of Events That Can Be Attended (LC 1353)
**Topic:** Greedy + Min-Heap (Day Sweep)
**Difficulty:** Medium

---

## Problem Statement

You are given an array of `events` where `events[i] = [startDay_i, endDay_i]`. Every event `i` starts at `startDay_i` and ends at `endDay_i`.

You can attend an event on any day `d` where `startDay_i <= d <= endDay_i`. You can only attend **one event per day**.

Return the **maximum number of events** you can attend.

**Example 1:**
```
Input: events = [[1,2],[2,3],[3,4]]
Output: 3
```

**Example 2:**
```
Input: events = [[1,2],[2,3],[3,4],[1,2]]
Output: 4
```

**Constraints:**
- `1 <= events.length <= 10^5`
- `1 <= startDay_i <= endDay_i <= 10^5`

---

## Conversation Log

**Interviewer:** Ready when you are. Please note the current time and share it.

**Aayush:** 10:01

**Interviewer:** Any clarifying questions before you start thinking about an approach?

**Aayush:** Can all the events have same start and ends?

**Interviewer:** Yes — multiple events can have the same startDay and endDay, including being fully identical.

**Aayush:** We can greedily choose the event which has least end time so that we can choose the ones with larger end times at later days.

**Interviewer:** Good intuition, but events have a range of valid days, not a single fixed day. Walk me through how you'd actually iterate.

**Aayush:** We sort the events array based on end times, initially current day is 1, and at each i if s[i]<=current_day<=e[i] we can attend this event. Record this event as attended on current_day and increment current_day.

**Interviewer:** Try this input: events = [[1,1],[1,4],[4,4]]. Walk through it.

**Aayush:** The algorithm will output 3 and the answer is 3.

**Interviewer:** Let me re-trace — your algorithm actually outputs 2, but optimal is 3 (day 1: [1,1], day 2: [1,4], day 4: [4,4]). current_day jumps to 3 after attending [1,4], but [1,4] could've been attended on day 2, 3, or 4. How would you fix this?

**Aayush:** We sort the events by end time. Then iterate L2R. Maintain a maxHeap of end times of chosen events. For event i, compare e[i] with max end time. If e[i] > max, attend. Else if e[i] <= max, attend only if maxH.size() + 1 <= maxH.top().

**Interviewer:** You're ignoring start days entirely. Try events = [[2,2],[2,2],[1,3]].

**Aayush:** 2 events and my algorithm also gives 2 since the second event [2,2] will not be included in maxHeap.

**Interviewer:** Re-trace: sorted = [[2,2],[2,2],[1,3]]. i=0 add. i=1: e=2, max=2, size+1=2<=2 ✓ add. i=2: e=3>2 add. Output 3, but optimal is 2. Your condition assumes days 1..max_end all available, but starts restrict that. What data structure helps you pick the right event on day d?

**Aayush:** Need hint.

**Interviewer:** Flip your perspective. Iterate over days, not events. On each day d, what events are in play (start<=d, end>=d), and which one should you pick?

**Aayush:** Sort the events by start times. Keep a min heap of end times. Start current_day = 1. Get available events (start<=day, end>=day), put them in heap. Consume event with minimal end time, increment day. If heap becomes empty but events remain, set day = start of current event.

**Interviewer:** Correct approach. What's termination condition and upper bound on day?

**Aayush:** Day can be max end time. Termination: while events remain or heap not empty.

**Interviewer:** Code it up.

**Aayush:** [Submits first version using `if(minH.empty()) return 0` instead of jumping current_day]

**Interviewer:** Trace events = [[1,1],[5,5]]. What does it output, what should it?

**Aayush:** Fixed all bugs — the issue was current_day was not being set properly.

```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<vector<int>> events {{2,2},{2,2},{1,3}};
    int n = events.size();
    sort(events.begin(),events.end());
    priority_queue<int,vector<int>,greater<int>> minH;
    int i=0, current_day=1, cnt=0;

    while(i<n || !minH.empty()) {
        if(minH.empty()) current_day = events[i][0];
        while(i<n && events[i][0] <= current_day) {
            minH.push(events[i][1]); i++;
        }
        while(!minH.empty() && minH.top() < current_day) minH.pop();
        if(!minH.empty()) {
            minH.pop(); current_day++; cnt++;
        }
    }
    cout<<"ans is "<<cnt<<endl;
}
```

TC: O(n log n), SC: O(n).

**Interviewer:** Correct. Be explicit each event enters/leaves heap at most once. Edge cases?

**Aayush:** No events, single event, all events on same day.

**Interviewer:** Does your algorithm depend on day values being small?

**Aayush:** The algorithm does not depend on endTime, just on number of events. So endDay can be up to 10^9.

**Interviewer:** Correct.

---

## Solution

**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<vector<int>> events {{2,2},{2,2},{1,3}};
    int n = events.size();
    sort(events.begin(),events.end());
    priority_queue<int,vector<int>,greater<int>> minH;
    int i=0, current_day=1, cnt=0;

    while(i<n || !minH.empty()) {
        if(minH.empty()) current_day = events[i][0];
        while(i<n && events[i][0] <= current_day) {
            minH.push(events[i][1]); i++;
        }
        while(!minH.empty() && minH.top() < current_day) minH.pop();
        if(!minH.empty()) {
            minH.pop(); current_day++; cnt++;
        }
    }
    cout<<"ans is "<<cnt<<endl;
}
```

**Time Complexity:** O(n log n)
**Space Complexity:** O(n)

---

## Feedback Given

### Scoring Rubric

| Category | Score (/5) | Notes |
|---|---|---|
| Problem Understanding & Clarification | 2.5 | Only one clarifying question. Didn't ask about constraint bounds, ties, integer days, or conflicts. |
| Approach & Thought Process | 2.5 | Two wrong approaches before correct one. First was classic interval-scheduling reflex ignoring range nature. Second was misapplied Course Schedule III pattern ignoring starts. Needed hint to reach day-sweep min-heap. |
| Code Quality & Correctness | 3.5 | Final code clean and correct. First version had early-termination bug caught only after counter-example. |
| Complexity Analysis | 4 | Correct O(n log n)/O(n). Good reasoning on day-range independence. Loose framing on heap ops. |
| Communication | 3 | Confidently proposed wrong approaches. Incorrectly claimed algorithm output "2" when it output "3". Shallow edge-case enumeration. |

**Overall: ~3/5**

### Key Takeaways
1. Don't anchor on patterns prematurely — sweep over days, not events.
2. Trace your own code before declaring correctness.
3. Front-load clarifying questions on constraints.
4. 65 min for a medium is long; target ~40 min.

**Time Taken: 65 minutes**
