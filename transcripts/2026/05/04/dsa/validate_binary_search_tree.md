# DSA Round Transcript
**Date:** 2026-05-04
**Start Time:** 13:17
**End Time:** 13:46
**Duration:** 29 minutes
**Problem:** Validate Binary Search Tree
**Topic:** Binary Trees / Recursion
**Difficulty:** Medium

---

## Problem Statement
Given the root of a binary tree, determine if it is a valid BST.
- Left subtree contains only nodes with keys strictly less than the node's key.
- Right subtree contains only nodes with keys strictly greater than the node's key.
- Both subtrees must also be valid BSTs.

**Example 1:** `[2,1,3]` → true
**Example 2:** `[5,1,4,null,null,3,6]` → false (3 in right subtree of 5 but 3 < 5)

**Constraints:** 1 to 10^4 nodes, values in [-2^31, 2^31 - 1].

---

## Conversation Log

**Interviewer:** Presented problem, asked for clarifying questions and approach.

**Aayush:** Recursively verify: each node > left child and < right child, and subtrees valid.

**Interviewer:** Pushed back with Example 2 — at node 4, both child checks pass, but the tree is invalid because 3 < 5 (root). What's missing?

**Aayush:** Inorder traversal — if the resulting list is sorted, valid BST.

**Interviewer:** Correct. Asked if list is needed or if O(1) extra possible.

**Aayush:** Just track inorder predecessor — each node > predecessor.

**Interviewer:** Asked him to code.

**Aayush (Attempt 1):** Used global prev/ans, but signature mismatch (`bool` returning void), `&` vs `&&`, no return ans.

**Interviewer:** Pointed out compile error, style, asked for early termination.

**Aayush (Attempt 2):** Fixed — `void` signature, global ans, early termination via `if (!root || !ans) return;`. Correct.

**Interviewer:** Traced Example 2, confirmed. Asked for complexity.

**Aayush:** Time O(N), space O(1).

**Interviewer:** Reminded about recursion stack — space is O(H).

**Interviewer:** Asked for an alternative approach.

**Aayush:** Min/max bounds — each node's left subtree has max bound = node val, min bound passed through. Right subtree has min bound = node val, max passed through.

**Interviewer:** Excellent — code it.

**Aayush (Attempt 1):** Multiple bugs — `node` vs `root` typos, leftover `prev` references from previous approach, syntax error (`:` vs `;`).

**Interviewer:** Listed bugs and pointed out critical edge case: with INT_MIN/INT_MAX bounds and `int` types, a node value of INT_MIN/INT_MAX would be incorrectly rejected.

**Aayush (Attempt 2):** Used LLONG_MIN/MAX in main but kept function signature as `int`.

**Interviewer:** Pointed out values get truncated to int at function boundary; signature must change too.

**Aayush:** "There would be long long for the value types"

**Interviewer:** Asked for actual code.

**Aayush (Attempt 3):** Same code with same bug.

**Interviewer:** Pointed out signature still says `int mn, int mx`.

**Aayush (Attempt 4 — Correct):** Changed signature to `long long mn, long long mx`. Working.

**Interviewer:** Confirmed correct. Noted `ans = ans && false` is just `ans = false`. Asked for complexity.

**Aayush:** Time O(N), space O(1).

**Interviewer:** Same recursion stack mistake — space is O(H). Second time in this round.

---

## Solution
**Aayush's Final Solution (Min/Max Bounds):**
```cpp
bool ans = true;

void inorder(TreeNode* root, long long mn, long long mx) {
    if (!root || !ans) return;
    inorder(root->left, mn, root->val);
    if (root->val >= mx || root->val <= mn) {
        ans = false;
        return;
    }
    inorder(root->right, root->val, mx);
}

// caller:
long long mn = LLONG_MIN, mx = LLONG_MAX;
inorder(root, mn, mx);
```

**Aayush's Inorder Predecessor Solution:**
```cpp
TreeNode* prev = NULL;
bool ans = true;

void inorder(TreeNode* root) {
    if (!root || !ans) return;
    inorder(root->left);
    if (prev != NULL) {
        ans = ans && (root->val > prev->val);
    }
    prev = root;
    inorder(root->right);
}
```

**Time Complexity:** O(N) — visit each node once.
**Space Complexity:** O(H) — recursion stack; worst case O(N) for skewed tree, O(log N) balanced.

---

## Feedback Given

| Category | Score | Notes |
|---|---|---|
| Problem Understanding & Clarification | 3/5 | No clarifying questions; missed value-range edge case. |
| Approach & Thought Process | 4.5/5 | Two correct approaches independently derived. |
| Code Quality & Correctness | 2.5/5 | Compile errors, leftover code from previous approach, type-truncation bug took 3 iterations. |
| Complexity Analysis | 2.5/5 | Said O(1) space twice — forgot recursion stack both times. |
| Communication | 4/5 | Clear, responsive. |

**Overall: ~3.3/5**

### Patterns to fix
1. Read full function signature when changing types — param types matter.
2. Recursion = at least O(H) space, never O(1).
3. Dry-run on example before submitting.
4. Clean up artifacts when switching approaches (leftover `prev` reference).

**Time Taken: 29 minutes**
