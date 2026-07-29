# DSA Round Transcript
**Date:** 2026-05-06
**Start Time:** 12:13
**End Time:** 12:38
**Duration:** 25 minutes
**Problem:** Minimum Number of Refueling Stops
**Topic:** Greedy + Heap
**Difficulty:** Hard

---

## Problem Statement
A car travels from a starting position to a destination which is `target` miles east of the starting position.

There are gas stations along the way. The gas stations are represented as an array `stations` where `stations[i] = [position_i, fuel_i]` indicates that the i-th gas station is `position_i` miles east of the starting position and has `fuel_i` liters of gas.

The car starts with an infinite tank of gas, which initially has `startFuel` liters of fuel in it. It uses 1 liter of gas per mile. When the car reaches a gas station, it may stop and refuel, transferring all the gas from the station into the car.

Return the minimum number of refueling stops the car must make in order to reach its destination. If it cannot reach the destination, return -1.

Constraints:
- 1 <= target, startFuel <= 10^9
- 0 <= stations.length <= 500
- 0 <= position_i < target
- 1 <= fuel_i <= 10^9
- Stations are sorted by position in strictly increasing order.

**Examples:**
- target=100, startFuel=10, stations=[[10,60],[20,30],[30,30],[60,40]] → 2
- target=1, startFuel=1, stations=[] → 0
- target=100, startFuel=1, stations=[[10,100]] → -1

---

## Conversation Log

**Interviewer:** Initially presented "Longest Substring with At Most K Distinct Characters."
**Aayush:** Solved this problem before.
**Interviewer:** Swapped to Minimum Number of Refueling Stops. Asked for start time.
**Aayush:** 12:13.
**Interviewer:** Asked for clarifying questions and approach.
**Aayush:** At each station the car has the option to stop or not. So we can do a recursion where we either stop at a station, refuel, and then continue, or we don't stop at a station and continue. We take the minimum of the 2 and return it. Base case: if no fuel and car not at station or destination then can't answer so return infinity. OR if no stations left and fuel < distance left from destination.
**Interviewer:** Pushed on state, complexity, and edge cases. Asked for the recursive function signature.
**Aayush:** solve(curPosition, station_index, target, fuelLeft).
**Interviewer:** Pointed out curPosition may be redundant with station_index, and that fuelLeft can be huge (up to 5*10^11), so memoization is infeasible. Hinted at greedy: "you've passed some stations without stopping; you're now stuck. Which one do you wish you'd stopped at?"
**Aayush:** What are the constraints?
**Interviewer:** Provided constraints. Restated greedy hint.
**Aayush:** Any hints?
**Interviewer:** Gave the "defer the decision" hint — toss passed stations into a bag, refuel retroactively when about to run out. Asked which data structure.
**Aayush:** Max heap of station fuels.
**Interviewer:** Confirmed and described full algorithm. Asked him to code.
**Aayush:** Submitted C++ solution (see below). Stated O(N log N) time, O(N) space.
**Interviewer:** Traced solution, confirmed correctness, flagged integer overflow bug (`distancePossible` should be `long long`). Asked for end time.
**Aayush:** 12:38.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<vector<int>> stations{{10,100}};
    int target = 100;
    int startFuel = 1;

    stations.push_back({target, 0});
    int distancePossible = startFuel;
    int ans = -1;
    int stops = 0;
    priority_queue<int> maxH;

    for(int i=0;i<stations.size();i++)
    {
        auto vec = stations[i];
        int position = vec[0];
        int fuel = vec[1];

        while(!maxH.empty() && distancePossible < position)
        {
            distancePossible += maxH.top();
            stops++;
            maxH.pop();
        }
        if(distancePossible < position)
        {
            cout<<"ans is "<<ans;
            return 0;
        }
        maxH.push(fuel);
    }

    cout<<"ans is "<<stops;
    return 0;
}
```

**Optimal Solution (same approach, with overflow fix):**
```cpp
int minRefuelStops(int target, int startFuel, vector<vector<int>>& stations) {
    stations.push_back({target, 0});
    long long distancePossible = startFuel;
    int stops = 0;
    priority_queue<int> maxH;

    for (auto& s : stations) {
        int position = s[0], fuel = s[1];
        while (!maxH.empty() && distancePossible < position) {
            distancePossible += maxH.top();
            maxH.pop();
            stops++;
        }
        if (distancePossible < position) return -1;
        maxH.push(fuel);
    }
    return stops;
}
```

**Time Complexity:** O(N log N)
**Space Complexity:** O(N)

---

## Feedback Given

**Problem Understanding & Clarification: 3/5**
- Jumped into approach before asking about constraints. Constraints determine whether DP/memo is even feasible — asking upfront would have saved time. Recurring pattern.

**Approach & Thought Process: 3.5/5**
- Started reasonably with recursion (stop/skip), correctly identified base cases conceptually.
- Didn't see the greedy "defer the decision" insight on his own — needed strong hint to reach max-heap pattern.
- Once hinted, locked onto max-heap quickly.

**Code Quality & Correctness: 4/5**
- Clean, readable code. Smart sentinel trick with `{target, 0}`.
- Logic is correct.
- **Bug: integer overflow.** `distancePossible` should be `long long` since cumulative fuel can hit 5*10^11. Didn't trace code against constraints — recurring weakness.

**Complexity Analysis: 5/5**
- Stated both time and space proactively and correctly. O(N log N) time, O(N) space.

**Communication: 4/5**
- Clear walk-through. Asked for constraints late. Could be more proactive about trade-offs before requesting hints.

**Overall: 19.5/25**
**Time Taken: 25 minutes**

**Key takeaways:**
1. Ask for constraints *first*. They eliminate entire approach families before you waste time.
2. After coding, mentally validate value ranges against variable types.
3. Greedy + heap "defer the decision" pattern (LeetCode 502, 1642, 1801).
