# DSA Round Transcript
**Date:** 2026-05-21
**Start Time:** 9:18
**End Time:** 10:28
**Duration:** 70 minutes
**Problem:** Decode Ways
**Topic:** Dynamic Programming
**Difficulty:** Medium

---

## Problem Statement
A message containing letters `A-Z` is encoded to numbers using this mapping:
```
'A' -> "1"
'B' -> "2"
...
'Z' -> "26"
```

To decode an encoded message, all the digits must be grouped, then mapped back into
letters using the reverse of the mapping above (there may be multiple ways). For
example, `"11106"` can be mapped into `"AAJF"` (1 1 10 6) or `"KJF"` (11 10 6).
The grouping `(1 11 06)` is invalid because `"06"` cannot be mapped into a letter.

Given a string `s` containing only digits, return the number of ways to decode it.

**Examples:**
- `s = "12"` -> `2`  ("AB" = 1 2, "L" = 12)
- `s = "226"` -> `3`  ("BZ", "VF", "BBF")
- `s = "06"` -> `0`  (invalid leading zero)

**Constraints:**
- `1 <= s.length <= 100`
- `s` contains only digits and may contain leading zero(s).

---

## Conversation Log

**Interviewer:** Presented the Decode Ways problem with examples. Asked Aayush to note the start time and raise any clarifying questions.

**Aayush:** 9:18 (start time).

**Aayush:** What are the constraints?

**Interviewer:** `1 <= s.length <= 100`; `s` contains only digits and may contain leading zero(s).

**Aayush:** If there are 2 neighbouring zeros then the string becomes invalid.

**Interviewer:** Correct — a `"0"` can only be the second digit of `"10"` or `"20"`, so `"00"` can never resolve. Asked for the full approach.

**Aayush:** Proposed a recursive solution `solve(i, last)` where `i` is the current index and `last` is the position of the last cut. At each index, choose to cut or not: can cut at `i` if `s[last+1..i] <= 26`; can skip a cut only if `i-last==1` and `s[i]!='0'`. Answer is the sum of both. Base cases: at `i==n-1` return whether `s[last+1..i] <= 26`; if `i-last==1` and `s[i]=='0'` return 0 (leading zero).

**Interviewer:** Asked two clarifying questions: (1) For a given `i`, what values can `last` take, and does that mean two parameters are really needed? (2) Does the `<= 26` check alone correctly reject the segment `"0"`?

**Aayush:** `last` can only be `i-1` or `i-2`. So `dp[i] = dp[i-1] + dp[i-2]`, and if `s[i]=='0'` then `dp[i] = dp[i-1]`. For zero handling, also ensure `s[last+1..i] > "0"`.

**Interviewer:** State simplification to 1D is right, but the recurrence is incomplete. Asked Aayush to dry-run it on `"27"` and `"10"`.

**Aayush:** Submitted a clean recursive (suffix) solution:
```cpp
int solve(string &s, int i) {
    if(i == s.size()) return 1;
    if(s[i] == '0') return 0;
    int ans = solve(s, i + 1);
    if(i + 1 < s.size()) {
        int num = (s[i] - '0') * 10 + (s[i + 1] - '0');
        if(num >= 10 && num <= 26) ans += solve(s, i + 2);
    }
    return ans;
}
```

**Interviewer:** Confirmed the recursion is correct; verified `"27"->1`, `"10"->1`, `"226"->3`, `"06"->0`, `"100"->0`. Asked for time and space complexity.

**Aayush:** Time `O(2^n)`, reducible to `O(n)` with memoization. Space `O(n)` for the recursion stack.

**Interviewer:** Correct. Asked Aayush to add memoization, and whether he could beat `O(n)` space.

**Aayush:** Submitted the `O(1)`-space bottom-up DP (see Solution below).

**Interviewer:** Confirmed it is the optimal solution; traced `"226"`, `"10"`, `"06"`, `"100"`, `"27"`. Asked for final complexity and end time.

**Aayush:** Time `O(N)`, space `O(1)`. End time 10:28.

---

## Solution
**Aayush's Final Solution:**
```cpp
int numDecodings(string s) {
    int n = s.size();
    int next = 1;       // dp[n]
    int next_next = 0;  // imaginary dp[n+1]

    for(int i = n - 1; i >= 0; i--) {
        int curr = 0;
        if(s[i] != '0') {
            curr = next;                       // single digit
            if(i + 1 < n) {                    // two digits
                int num = (s[i]-'0') * 10 + (s[i+1]-'0');
                if(num >= 10 && num <= 26) curr += next_next;
            }
        }
        next_next = next;
        next = curr;
    }
    return next;
}
```

**Optimal Solution (if different):** Same as Aayush's final solution — `O(N)` time, `O(1)` space bottom-up DP. This is optimal.

**Time Complexity:** O(N)
**Space Complexity:** O(1)

---

## Feedback Given

# Feedback — Decode Ways

## What went well
- Asked for constraints up front (a recurring past gap — improved this round).
- Spotted the `"00"` edge case proactively before coding.
- Jumped straight to the `O(1)`-space bottom-up DP when asked to optimize, rather than stopping at the memoized version.
- Complexity analysis was precise and correct at every step.
- Final code (both recursion and iterative DP) was clean and correct on all edge cases.

## What to work on
1. **First approach was over-engineered.** The `solve(i, last)` two-parameter cut formulation obscured the structure; it took a prompt to see `last ∈ {i-1, i-2}` collapses to a clean 1D DP. Pause before coding and ask: what is the minimal state?
2. **Condensed recurrence had two bugs.** `dp[i] = dp[i-1] + dp[i-2]` with `dp[i]=dp[i-1]` for `s[i]=='0'` — the `dp[i-2]` term must be gated on `10 <= value <= 26`, and the zero case should be `dp[i-2]` gated on a valid pair, not `dp[i-1]`. Recovered by switching approaches rather than catching the bug.
3. **Answered dry-run questions with code instead of words.** When the interviewer asks a direct question, answer it verbally first — it surfaces bugs before they hit the editor.
4. **Never traced his own final solution** — the interviewer verified edge cases. Build the habit of walking 2-3 cases yourself before declaring done.

## Scoring
| Dimension | Score |
|---|---|
| Problem Understanding & Clarification | 8 / 10 |
| Approach & Thought Process | 6.5 / 10 |
| Code Quality & Correctness | 9 / 10 |
| Complexity Analysis | 9.5 / 10 |
| Communication | 6.5 / 10 |

**Overall: 7.9 / 10** — Solid. Landed the optimal solution with optimal space. Points lost were all in process: a cleaner initial framing and verbal engagement with the interviewer.

**Time Taken: 70 minutes**
