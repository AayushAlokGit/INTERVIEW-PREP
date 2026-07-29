# DSA Round Transcript
**Date:** 2026-06-08
**Start Time:** 9:19
**End Time:** 9:51
**Duration:** 32 minutes
**Problem:** Construct Binary Tree from Preorder and Inorder Traversal
**Topic:** Trees / Recursion / Divide & Conquer
**Difficulty:** Medium

---

## Problem Statement
Given two integer arrays `preorder` and `inorder` where `preorder` is the preorder traversal of a binary tree and `inorder` is the inorder traversal of the same tree, construct and return the binary tree.

**Example:**
```
preorder = [3, 9, 20, 15, 7]
inorder  = [9, 3, 15, 20, 7]

        3
       / \
      9   20
         /  \
        15   7
```

**Constraints:**
- 1 <= preorder.length <= 3000
- inorder.length == preorder.length
- -3000 <= preorder[i], inorder[i] <= 3000
- preorder and inorder consist of unique values
- inorder is a valid inorder traversal of the tree preorder represents

---

## Conversation Log

**Interviewer:** Presented the problem and asked for clarifying questions.

**Aayush:** "what are the constraints?"

**Interviewer:** Provided constraints (length up to 3000, unique values, valid pairing).

**Aayush:** Explained approach: 0th element of preorder is the root. Since values are unique, find its index in inorder. That index gives sizes of left and right subtrees. Right subtree's preorder is `[preorder.sz - rTreeSize, preorder.sz-1]`, right subtree's inorder is `[inorderRootIdx, inorder.sz-1]`. Similarly derive left subtree ranges. Create node with root value, recurse on left and right, return node.

**Interviewer:** Validated the core idea, then probed: right subtree inorder range `[inorderRootIdx, ...]` — does that include the right element?

**Aayush:** "+1 should be there" — corrected to `[inorderRootIdx+1, inorder.sz-1]`.

**Interviewer:** Asked how he'd find the root index in inorder each call and the complexity impact. Asked him to code it (chose C++).

**Aayush:** Submitted recursive `solve(preorder, inorder, preS, preE, inS, inE)` with linear search over inorder, computing `lTreeSize = lInE - lInS`, `lPreS = preS`, `lPreE = lPreS + lTreeSize`, base case `if(preE == preS) return root`.

**Interviewer:** Asked him to dry-run on the example with literal numbers and report the left child's value.

**Aayush:** Identified `lTreeSize` needs `+1` and `lPreE = lPreS + lTreeSize - 1`.

**Interviewer:** Continued the trace — with `lPreS = preS = 0`, the left call is `solve(0,0,...)` producing value 3 again. Asked what value the left child gets vs. what it should be.

**Aayush:** "lPreS = preS + 1" — corrected.

**Interviewer:** Confirmed example now reconstructs. Gave the `[1,2] / [2,1]` single-child tree and asked him to trace the empty right-subtree call.

**Aayush:** "handle the case of nulls when preE < preS" — identified the missing base case.

**Interviewer:** Confirmed `if (preS > preE) return nullptr;` guard at the top. Asked for time and space complexity.

**Aayush:** "time complexity is O(n) since we visit each node once. Space is O(h) for recursion stack."

**Interviewer:** Pointed out the linear search runs per call; asked for the real complexity.

**Aayush:** "O(n*n)" (corrected).

**Interviewer:** Asked for the optimization to O(n).

**Aayush:** "a hashmap of inorder value to its index; since values are unique the keys are unique. Space becomes O(h + n), time reduces to O(n)."

---

## Solution

**Aayush's Final Solution (after corrections):**
```cpp
class Solution {
    TreeNode* solve(vector<int>& preorder, vector<int>& inorder,
                    int preS, int preE, int inS, int inE) {
        if (preS > preE) return nullptr;            // fix 3: empty subtree guard
        TreeNode* root = new TreeNode(preorder[preS]);
        if (preE == preS) return root;

        int inorderRootIdx;
        for (int i = 0; i < inorder.size(); i++) {
            if (inorder[i] == root->val) { inorderRootIdx = i; break; }
        }

        int lInS = inS, lInE = inorderRootIdx - 1;
        int rInS = inorderRootIdx + 1, rInE = inE;
        int lTreeSize = lInE - lInS + 1;            // fix 1: +1
        int lPreS = preS + 1;                        // fix 2: preS + 1
        int lPreE = lPreS + lTreeSize - 1;
        int rPreS = lPreE + 1;
        int rPreE = preE;

        root->left  = solve(preorder, inorder, lPreS, lPreE, lInS, lInE);
        root->right = solve(preorder, inorder, rPreS, rPreE, rInS, rInE);
        return root;
    }
public:
    TreeNode* buildTree(vector<int>& preorder, vector<int>& inorder) {
        int n = preorder.size();
        return solve(preorder, inorder, 0, n - 1, 0, n - 1);
    }
};
```
This version is O(n²) due to the linear `inorder` search per call.

**Optimal Solution (O(n) — discussed, not coded):**
Precompute `unordered_map<int,int> idx` mapping each inorder value to its index (valid because values are unique). Replace the linear search with an O(1) lookup. Yields **O(n) time, O(n) space**.

**Time Complexity:** O(n²) as written → O(n) with hashmap optimization
**Space Complexity:** O(h) recursion stack (worst case O(n)); O(n) total with the hashmap

---

## Feedback Given

You landed on the optimal solution and reasoned through every correctness trap — but it took dry-run prompting to surface bugs sitting in plain sight. That's the recurring theme.

**What went well**
- Clarifying questions up front (asked constraints before coding); used the uniqueness fact to drive the hashmap optimization.
- Clean decomposition stated clearly before coding.
- Self-corrected every bug once pointed at the right spot, with understanding of why.

**What needs work**
- Shipped three off-by-one / boundary bugs without tracing: `lTreeSize` missing `+1`, `lPreS` should be `preS+1`, missing `preS > preE` null guard. A single dry-run before declaring done catches all three.
- Empty-subtree case (canonical trap for this problem) only found when handed the `[1,2]` tree.
- Asserted O(n) while code had an O(n) search inside every call (true O(n²)); fixed instantly when prompted.

**Scoring (out of 5)**
| Category | Score |
|---|---|
| Problem understanding & clarification | 4.5 |
| Approach & thought process | 4 |
| Code quality & correctness | 2.5 |
| Complexity analysis | 3 |
| Communication | 3.5 |

**Overall: ~3.5/5** — strong conceptual grasp and optimal endpoint, but correctness discipline (dry-running before declaring done) is the gap.

**The one habit to build:** Before saying "done," run your actual code on the given example with literal index numbers — not your intent, the code. Every bug today dies in that 60-second exercise.

**Time Taken: 32 minutes**
