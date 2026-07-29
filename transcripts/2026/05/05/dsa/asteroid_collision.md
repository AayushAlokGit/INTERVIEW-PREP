# DSA Round Transcript
**Date:** 2026-05-05
**Start Time:** 12:12
**End Time:** 12:33
**Duration:** 21 minutes
**Problem:** Asteroid Collision
**Topic:** Stack, Simulation
**Difficulty:** Medium

---

## Problem Statement

We are given an array `asteroids` of integers representing asteroids in a row.

For each asteroid, the absolute value represents its size, and the sign represents its direction (positive = right, negative = left). Each asteroid moves at the same speed.

Find out the state of the asteroids after all collisions. If two asteroids meet:
- The smaller one explodes.
- If both are the same size, both explode.
- Two asteroids moving in the same direction never meet.

**Examples:**
- `[5, 10, -5]` → `[5, 10]`
- `[8, -8]` → `[]`
- `[10, 2, -5]` → `[10]`
- `[-2, -1, 1, 2]` → `[-2, -1, 1, 2]`

**Constraints:**
- `2 <= asteroids.length <= 10^4`
- `-1000 <= asteroids[i] <= 1000`
- `asteroids[i] != 0`

---

## Conversation Log

**Interviewer:** Presented the problem with all four examples and constraints.

**Aayush:** "Iterate through the array and at each index i we need to know the last surviving asteroid and its direction. If same direction then current asteroid no effect. If opposite direction, while last element is opposite direction make collisions. O(n) time, O(n) space for stack."

**Interviewer:** Confirmed approach. Asked when collisions happen specifically.

**Aayush:** "If current < 0 and stack top > 0 then collisions happen."

**Interviewer:** Confirmed. Asked him to code.

**Aayush:** Submitted v1 — bugs:
- Output reversed (popping stack into ans).
- In else branch (current beats last), pushed `nums[i]` onto stack which halted the inner collision loop.
- When current is negative but top is non-positive (stack empty or top<0), current never gets pushed.

**Interviewer:** Walked through three failing traces: `[5,10,-5]` (order reversed), `[10,2,-5]` (push of -5 prevents further collision), `[-2,-1,1,2]` (-1 dropped).

**Aayush:** Submitted v2 — added `reverse(ans)`, removed errant `st.push(nums[i])` in else branch, added post-while `if(st.empty() || st.top()<0) st.push(nums[i])`.

**Interviewer:** Trace `[8,-8]`: same-size destroys both, but post-while sees empty stack and pushes `-8`. Output `[-8]` instead of `[]`. Fix: track `bool alive` flag set false in equal-size case.

**Aayush:** Submitted v3 — added `bool shouldInsert` flag set false on equal-size break. Final post-condition `if(shouldInsert && (st.empty() || st.top()<0)) st.push(nums[i])`.

**Interviewer:** Verified all four examples pass. Noted post-condition is doing double duty (also handles last>current case implicitly). Cleaner version: set `shouldInsert=false` in both destruction branches and reduce post-loop to `if(shouldInsert) st.push(nums[i])`.

**Aayush:** "12:33"

---

## Solution

**Aayush's Final Solution:**
```cpp
for(int i = 0; i < n; i++) {
    if(st.empty() || nums[i] > 0) {
        st.push(nums[i]);
        continue;
    }
    if(nums[i] < 0) {
        bool shouldInsert = true;
        while(!st.empty() && st.top() > 0) {
            int last = st.top();
            st.pop();
            if(abs(last) == abs(nums[i])) {
                shouldInsert = false;
                break;
            } else if(abs(last) > abs(nums[i])) {
                st.push(last);
                break;
            }
        }
        if(shouldInsert && (st.empty() || st.top() < 0))
            st.push(nums[i]);
    }
}
// reverse(ans) when popping for output
```

**Cleaner Version:**
```cpp
for(int x : nums) {
    bool alive = true;
    while(alive && !st.empty() && st.top() > 0 && x < 0) {
        if(st.top() < -x) { st.pop(); }
        else if(st.top() == -x) { st.pop(); alive = false; }
        else { alive = false; }
    }
    if(alive) st.push(x);
}
```

**Time Complexity:** O(n) — each element pushed/popped at most once.
**Space Complexity:** O(n) — stack.

---

## Feedback Given

### Scoring

| Category | Score (1-5) | Notes |
|---|---|---|
| Problem Understanding & Clarification | 4 | Identified collision conditions correctly. Didn't ask clarifying questions. |
| Approach & Thought Process | 4.5 | Stack approach identified immediately. Correct insight on collision conditions. |
| Code Quality & Correctness | 2.5 | Three iterations. Recurring: not tracing edge cases pre-submit, missing same-size-destroys-both, missing negative-after-non-positive-top. Final code works but convoluted. |
| Complexity Analysis | 5 | Stated O(n)/O(n) upfront with correct reasoning. |
| Communication | 4 | Clear and concise. |

**Overall: 4 / 5**

### Time Taken: 21 minutes

### Key Takeaways

1. **Trace edge cases before declaring done.** Missed `[8,-8]` (mutual destruction) and `[-2,-1]` (negative after negative). Make a habit of testing minimal-size inputs covering each branch.
2. **Prefer explicit flags over implicit guards.** Final post-while condition implicitly handles the "current destroyed by larger last" case. A single `alive` flag in all destruction branches is clearer.
3. **`if/continue` early-return for easy cases is a good pattern.** Keep using it.
