# DSA Round Transcript
**Date:** 2026-05-18
**Start Time:** 9:38
**End Time:** 10:24
**Duration:** 46 minutes
**Problem:** Basic Calculator II
**Topic:** Stack / String Parsing
**Difficulty:** Medium

---

## Problem Statement
Given a string `s` which represents an arithmetic expression, evaluate it and return the result.

Integer division truncates toward zero. The expression contains only non-negative integers, operators `+`, `-`, `*`, `/`, and spaces. No `eval()` allowed.

### Examples
- `"3+2*2"` → `7`
- `" 3/2 "` → `1`
- `" 3+5 / 2 "` → `5`

### Constraints
- `1 <= s.length <= 3 * 10^5`
- Integers in `[0, 2^31 - 1]`
- Expression is valid; result fits in 32-bit integer.

---

## Conversation Log

**Interviewer:** Presented problem. Asked for clarifying questions.

**Aayush:** Can there be multi-digit operands?

**Interviewer:** Yes — confirmed range up to 2^31 - 1.

**Aayush:** Proposed tokenizing into a vector of strings (operands + operators), then Shunting Yard to convert to postfix, then evaluate postfix with a stack.

**Interviewer:** Correct but heavy. Pushed on (1) complexity, (2) simpler single-pass since no parentheses.

**Aayush:** O(n) time, O(n) space for both passes.

**Interviewer:** Hinted at one-stack approach: handle `+`/`-` by pushing signed operand; on `*`/`/`, pop top, combine with next operand, push back.

**Aayush:** Articulated the approach correctly: push for `+`/`-`, pop-combine-push for `*`/`/`, sum stack at end.

**Interviewer:** Traced `"3+2*2"` → 7. Asked to code.

**Aayush:** Wrote C++ solution (v1) using a `sign` flag that flipped on `-`. Claimed empty-string was handled.

**Interviewer:** Traced `"3-2*2"` → produced 7 instead of -1. Traced `"3-2+5"` → produced -4 instead of 6. Identified bug: `sign` never reset, plus double-applied in `*//` branch. Also flagged `size_t` underflow on empty input.

**Aayush:** Fixed by resetting `sign = 1` after each push or `*//` operation. Submitted v2.

**Interviewer:** Traced multiple cases (`"3-2*2"`, `"3-2+5"`, `"6-4/2"`, `"2*3-4*5+6"`) — all correct. Noted empty-string underflow still latent. Asked for complexity.

**Aayush:** O(n) time, O(n) space.

**Interviewer:** Mentioned O(1) space optimization possible (running sum + last term). Wrapped up.

---

## Solution

**Aayush's Final Solution (C++):**
```cpp
#include <bits/stdc++.h>
using namespace std;

bool isOperator(char c) { return (c=='*'||c=='/'||c=='+'||c=='-'); }
bool isOperator(string c) { return (c=="*"||c=="/"||c=="+"||c=="-"); }

int main()
{
    string s = " 3-5 / 2 ";
    int n = s.size();
    vector<string> infix;

    for (int i = 0; i < n; i++) {
        if (s[i] == ' ') continue;
        if (isdigit(s[i])) {
            string operand = "";
            while (i < n && isdigit(s[i])) { operand.push_back(s[i]); i++; }
            infix.push_back(operand);
            i--;
        } else if (isOperator(s[i])) {
            infix.push_back(string(1, s[i]));
        }
    }

    stack<long long> st;
    long long sign = 1;
    int i;
    for (i = 0; i < infix.size() - 1; i++) {
        if (infix[i] == "-") {
            sign *= -1;
        } else if (infix[i] == "*" || infix[i] == "/") {
            long long op1 = st.top(); st.pop();
            long long op2 = stoll(infix[i+1]);
            long long res = (infix[i] == "*") ? op1*op2 : op1/op2;
            i++;
            st.push(res * sign);
            sign = 1;
        } else if (infix[i] != "+") {
            st.push(stoll(infix[i]) * sign);
            sign = 1;
        }
    }
    if (i == infix.size() - 1) st.push(stoll(infix[i]) * sign);

    long long ans = 0;
    while (!st.empty()) { ans += st.top(); st.pop(); }
    cout << "ans is " << ans << endl;
    return 0;
}
```

**Optimal Solution (single pass, cleaner, C++):**
```cpp
int calculate(string s) {
    stack<int> st;
    int num = 0;
    char prev_op = '+';
    for (int i = 0; i < (int)s.size(); i++) {
        char c = s[i];
        if (isdigit(c)) num = num * 10 + (c - '0');
        if ((!isdigit(c) && c != ' ') || i == (int)s.size() - 1) {
            if (prev_op == '+') st.push(num);
            else if (prev_op == '-') st.push(-num);
            else if (prev_op == '*') { int t = st.top(); st.pop(); st.push(t * num); }
            else if (prev_op == '/') { int t = st.top(); st.pop(); st.push(t / num); }
            prev_op = c;
            num = 0;
        }
    }
    int ans = 0;
    while (!st.empty()) { ans += st.top(); st.pop(); }
    return ans;
}
```

**Time Complexity:** O(n)
**Space Complexity:** O(n) — can be reduced to O(1) by tracking running sum + last term.

---

## Feedback Given

### Scoring

| Category | Score | Notes |
|---|---|---|
| Problem Understanding & Clarification | 3/5 | Asked about multi-digit operands. Missed: division truncation semantics, empty input, overflow. |
| Approach & Thought Process | 3/5 | Defaulted to Shunting Yard. Needed hint to see the simpler no-parens single-stack approach. |
| Code Quality & Correctness | 2/5 | Two bugs in v1: sticky `sign`, double-applied `sign` in `*//`. Claimed empty-string handled — `size_t` underflow makes it unsafe. |
| Complexity Analysis | 4/5 | Correct O(n)/O(n). Didn't volunteer O(1) space follow-up. |
| Communication | 3/5 | Clear explanation. Thin edge case coverage — didn't volunteer mixed `-` and `*//` cases, which is exactly where the bug was. |

### What went well
- Clean tokenization
- Absorbed one-stack idea quickly
- Confident, correct complexity

### What to work on
- Trace before declaring done — `"3-2*2"` would have caught both bugs in 30 seconds.
- Enumerate edge cases up front (empty, single number, large numbers, division semantics).
- After first approach, ask: "is there a problem-specific shortcut?" before committing to a general framework.
- Practice merging two-pass into single-pass.

**Time Taken: 46 minutes** — on the long side for a medium; target ~30 minutes.
