# DSA Round Transcript
**Date:** 2026-05-21
**Start Time:** 10:46
**End Time:** 11:17
**Duration:** 31 minutes
**Problem:** Restore IP Addresses
**Topic:** Backtracking / Recursion
**Difficulty:** Medium

---

## Problem Statement
A valid IP address consists of exactly four integers separated by single dots.
Each integer is between 0 and 255 (inclusive) and cannot have leading zeros.

Given a string `s` containing only digits, return all possible valid IP addresses
that can be formed by inserting dots into `s`. Digits cannot be reordered or removed.

**Examples:**
- `s = "25525511135"` -> `["255.255.11.135", "255.255.111.35"]`
- `s = "0000"` -> `["0.0.0.0"]`
- `s = "101023"` -> `["1.0.10.23","1.0.102.3","10.1.0.23","10.10.2.3","101.0.2.3"]`

**Constraints:**
- `1 <= s.length <= 20`
- `s` consists of digits only.

---

## Conversation Log

**Interviewer:** Presented Restore IP Addresses with examples. Asked for start time, clarifying questions, and approach.

**Aayush:** What are the constraints?

**Interviewer:** `1 <= s.length <= 20`, digits only. Noted that a valid IP encodes at most 12 digits.

**Aayush:** Start time 10:46. Proposed `solve(i, global_ans, cur_ans)`: if `i==n` push `cur_ans`; if `s[i]=='0'` return (avoid leading zeros); for `len` 1..3, take `s[i:i+len]`, check in `[0,255]`, append and recurse.

**Interviewer:** Pressure-tested two points: (1) the base case doesn't track segment count, so "exactly 4 integers" isn't enforced — trace `"1111"`; (2) `if s[i]=='0' return` rejects valid `"0000"`.

**Aayush:** Corrected: need to track segment count, prune if `cnt > 4`, only record when `cnt == 4 && i == n`. For `s[i]=='0'`, the only legal segment is the single `"0"` (len forced to 1), then recurse.

**Interviewer:** Both corrections right. Asked Aayush to code it.

**Aayush:** Submitted a solution, but the `s[i]=='0'` branch did `cur.push_back("0"); solve(...); return;` with no `cur.pop_back()`.

**Interviewer:** Pointed at the discrepancy between the two branches and asked: after the recursive call in the zero branch returns, what state is `cur` in? Asked Aayush to trace `"1010"`.

**Aayush:** Argued no restore was needed: "when `s[i]=='0'` we do not need to restore `cur` because no other segment would be possible if the first element is 0."

**Interviewer:** Walked the full `"1010"` trace, showing the backtracking contract violation — the parent's `pop_back()` removes the wrong element and `"1"` leaks into later branches, producing 5-segment garbage like `"1.0.10.1.0"`. Explained the contract: when `solve` returns, `cur` must equal what it was on entry — for the caller, not the siblings.

**Aayush:** Fixed it by adding `cur.pop_back()` in the zero branch.

**Interviewer:** Confirmed correct. Noted the `main` harness prints a trailing dot; real return value should be properly dot-joined. Asked for time and space complexity.

**Aayush:** Time `O(nC3)` (choosing 3 dot positions); space `O(n)` recursion stack + `O(nC3)` answer.

**Interviewer:** Probed both: recursion depth is bounded by the segment count (does it grow with `n`?); and dots can't be placed freely since each segment is at most 3 digits.

**Aayush:** Recursion stack max depth is 4; time is `O(3^4)`.

**Interviewer:** Both correct. Wrapped up. End time 11:17.

---

## Solution
**Aayush's Final Solution:**
```cpp
void solve(string &s, int i, int cnt, vector<vector<string>> &ans, vector<string> &cur)
{
    if(i == s.size()) {
        if(cnt == 4) ans.push_back(cur);
        return;
    }
    if(cnt > 4) return;

    if(s[i] == '0') {
        cur.push_back("0");
        solve(s, i+1, cnt+1, ans, cur);
        cur.pop_back();
        return;
    }

    for(int len = 1; len <= 3; len++) {
        if(i + len - 1 < s.size()) {
            string str = s.substr(i, len);
            int num = stoi(str);
            if(num >= 0 && num <= 255) {
                cur.push_back(str);
                solve(s, i+len, cnt+1, ans, cur);
                cur.pop_back();
            }
        }
    }
}
```

**Optimal Solution (if different):** Same approach — standard backtracking. Optimal.

**Time Complexity:** O(3^4) partitions explored — constant branching; O(n) work per result to build strings.
**Space Complexity:** O(1) recursion stack (max depth 4); output not counted as auxiliary.

---

## Feedback Given

# Feedback — Restore IP Addresses

## What went well
- Asked for constraints up front.
- Final code was clean and correct once fixed — leading zeros, exactly-4-segments, bounds all handled.
- Eventually nailed the complexity: constant depth 4, O(3^4) partitions.

## What to work on
1. **Initial approach missed two core constraints** — no segment count (didn't enforce exactly 4 integers) and a blanket `if s[i]=='0' return` that would kill valid inputs like `"0000"`. List a problem's hard constraints before coding and ensure the state encodes each.
2. **Shipped a backtracking bug** — missing `cur.pop_back()` in the zero branch, violating the contract that shared state must be restored on return.
3. **Defended the bug instead of tracing it** — when explicitly asked to trace `"1010"`, responded with a wrong rationale instead of running the trace. When the interviewer asks you to trace, trace. The trace is the proof; intuition is just the hypothesis.
4. **Both complexity bounds were wrong initially** — said `O(n)` recursion stack (it's constant 4) and `O(nC3)` time (it's `O(3^4)`, constant, since segments are capped at 3 digits). Corrected only after probing.

## Scoring
| Dimension | Score |
|---|---|
| Problem Understanding & Clarification | 7 / 10 |
| Approach & Thought Process | 5.5 / 10 |
| Code Quality & Correctness | 5 / 10 |
| Complexity Analysis | 5.5 / 10 |
| Communication | 5 / 10 |

**Overall: 5.6 / 10** — Below the first round. The algorithm wasn't the issue; verification discipline was: an unrestored-state bug, defending it instead of tracing, and two loose complexity bounds. Highest-leverage fix: when asked to trace, trace — don't assert.

**Time Taken: 31 minutes**
