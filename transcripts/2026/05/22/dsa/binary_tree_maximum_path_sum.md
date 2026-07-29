# DSA Round Transcript
**Date:** 2026-05-22
**Start Time:** 9:21
**End Time:** 9:45
**Duration:** 24 minutes
**Problem:** Binary Tree Maximum Path Sum
**Topic:** Binary Trees / Recursion (post-order DFS)
**Difficulty:** Hard

---

## Problem Statement
A **path** in a binary tree is a sequence of nodes where each pair of adjacent nodes in the sequence has an edge connecting them. A node can appear in the sequence at most once. The path does not need to pass through the root.

The **path sum** of a path is the sum of the node values in that path.

Given the `root` of a binary tree, return the maximum path sum of any non-empty path.

**Example 1**
```
       1
      / \
     2   3

Input:  root = [1,2,3]
Output: 6
Explanation: Optimal path 2 -> 1 -> 3 = 6.
```

**Example 2**
```
      -10
      /  \
     9    20
         /  \
        15   7

Input:  root = [-10,9,20,null,null,15,7]
Output: 42
Explanation: Optimal path 15 -> 20 -> 7 = 42.
```

Constraints discussed: tree has at least one node; node values may be negative/zero/positive (range ~[-1000, 1000]); up to ~3 x 10^4 nodes; the maximum path sum can itself be negative (all-negative tree).

---

## Conversation Log

**Interviewer:** Presented the problem and asked for clarifying questions / approach. Asked Aayush to note start time.

**Aayush:** Start time 9:21.

**Aayush:** Can the tree be empty?

**Interviewer:** No — the tree has at least one node. Also noted node values can be negative/zero/positive (~[-1000,1000]) and up to ~3x10^4 nodes.

**Aayush:** Can the maximum path sum be negative?

**Interviewer:** Yes — if all nodes are negative, the answer is the single least-negative node.

**Aayush:** We traverse in post-order. At each node we return to the parent the maximum path sum from this node going up — three options: just the node, node + left subtree chain, node + right subtree chain; we take the max and return it. Separately, at each node we compute a local max path sum, and to update the global max we also consider the path left subtree + node + right subtree.

**Interviewer:** Confirmed the decomposition is correct — return value is the single upward chain (3 candidates), global update additionally considers the split path. Asked: when a child's subtree returns a negative value, what do you do with it for (a) the return value and (b) the global update?

**Aayush:** When returning to parent we always return max(node value, lTree + node.val, rTree + node.val), so a negative return is simply not chosen by the max. The global update also chooses the maximum of all possible values.

**Interviewer:** Correct. Asked Aayush to code it; provided C++ boilerplate.

**Aayush:** Submitted C++ solution (see below).

**Interviewer:** Asked Aayush to (1) re-read the public `maxPathSum` function as a compiler would, and (2) dry-run `solve` on Example 2.

**Aayush:** Noted TreeNode is not defined.

**Interviewer:** Clarified TreeNode is provided by the judge; pointed at the three lines inside `maxPathSum`.

**Aayush:** `global_ans` should be passed and returned (instead of `ans`).

**Interviewer:** Confirmed the naming-bug fix. Asked for the dry-run on Example 2.

**Aayush:** Start from -10, go to 9, return 9 (global_ans=9). Go to 20, go to 15, return 15 (global_ans=15). Go to 7, return 7. At 20 return 35, global_ans=42. At -10 return 25.

**Interviewer:** Confirmed global_ans ends at 42 — correct. Asked for time and space complexity.

**Aayush:** Time O(number of nodes) — each node visited once. Space O(1) for variables plus O(height of tree) for recursion stack.

**Interviewer:** Asked for the worst-case height and resulting worst-case space.

**Aayush:** Worst case is a skewed tree, height = n.

**Interviewer:** Confirmed worst case O(n), balanced O(log n). Asked whether further optimization is possible.

**Aayush:** It is already optimal.

**Interviewer:** Asked him to justify why O(n) time is a hard lower bound.

**Aayush:** Because we must explore all nodes to arrive at the truly global answer.

**Interviewer:** Confirmed. Asked for end time.

**Aayush:** End time 9:45.

---

## Solution
**Aayush's Final Solution (C++):**
```cpp
class Solution {
public:
    int maxPathSum(TreeNode* root) {
        int global_ans = INT_MIN;
        solve(root, global_ans);   // fixed: was solve(root, ans)
        return global_ans;          // fixed: was return ans
    }

private:
    int solve(TreeNode* node, int &global_ans) {
        if(!node) return 0;

        int lTree = solve(node->left, global_ans);
        int rTree = solve(node->right, global_ans);

        int cur_ans = INT_MIN;
        cur_ans = max(cur_ans, node->val);
        cur_ans = max(cur_ans, lTree + node->val);
        cur_ans = max(cur_ans, rTree + node->val);

        global_ans = max(global_ans, cur_ans);
        global_ans = max(global_ans, lTree + node->val + rTree);

        return cur_ans;
    }
};
```

**Optimal Solution:** Same as above — this is the standard optimal approach. (A common stylistic variant clamps each child's return with `max(0, childReturn)` so only one combined candidate `node + L + R` is needed for the global update; Aayush's explicit enumeration of all candidates is equally correct.)

**Time Complexity:** O(n) — each node visited exactly once.
**Space Complexity:** O(h) recursion stack — O(n) worst case (skewed tree), O(log n) balanced.

---

## Feedback Given

# Feedback — Binary Tree Maximum Path Sum (Hard)

**Time Taken: 24 minutes**

## Overall
Strong round. This is a LeetCode Hard, and Aayush arrived at the correct, optimal solution without a single algorithmic hint — the only nudges were code-review prompts. The hard part of this problem is the conceptual split between "what I return to my parent" vs. "what I use to update the global answer," and he articulated that distinction himself, up front, before writing any code. Senior-level pattern recognition.

## What went well
- Clarifying questions — proactively asked about the empty tree and whether the answer can be negative. Both are exactly the questions that matter here.
- The core insight — correctly identified that the return value is a single upward chain (3 candidates) while the global update additionally considers the split path (L + node + R).
- Complexity analysis — precise and complete. Named the recursion stack as a space cost (not just "O(1)"), correctly expressed worst case as O(n) for a skewed tree, and gave a valid lower-bound argument for O(n) time. Historically a soft spot — clean today.

## What to sharpen
1. Self-review your code before saying you're done. Declared `global_ans` but passed/returned `ans` — a hard compile error. Did not catch it independently; needed narrowing to three specific lines. Build the habit: read every line as a compiler would, then trace one example — did neither unprompted.
2. Clarify value ranges yourself. Asked great semantic questions but never asked about node value ranges or tree size — interviewer volunteered those. Ranges drive overflow decisions.
3. Justify claims on first pass. "I believe it's optimal" — true, but had to be asked for the why. Volunteer reasoning unprompted.

## Scoring

| Criterion | Score | Notes |
|---|---|---|
| Problem understanding & clarification | 7/10 | Good semantic questions; skipped value ranges |
| Approach & thought process | 9/10 | Nailed the return-vs-global split independently |
| Code quality & correctness | 7/10 | Logic correct first try; uncaught compile bug |
| Complexity analysis | 9/10 | Precise, complete, with lower-bound argument |
| Communication | 7/10 | Clear traces; slightly terse, needed prompting to justify |

**Overall: 39/50** — a strong, hire-leaning performance on a Hard problem.
