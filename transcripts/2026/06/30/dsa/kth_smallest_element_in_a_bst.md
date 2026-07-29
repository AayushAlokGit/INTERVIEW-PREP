# DSA Round Transcript
**Date:** 2026-06-30
**Start Time:** 10:32
**End Time:** 11:16
**Duration:** 44 minutes
**Problem:** Kth Smallest Element in a BST
**Topic:** Trees / BST
**Difficulty:** Medium

---

## Problem Statement
Given the `root` of a binary search tree and an integer `k`, return the k-th smallest value (1-indexed) of all the node values in the tree.

**Example 1:**
```
        3
       / \
      1   4
       \
        2
k = 1  ->  Output: 1
```

**Example 2:**
```
            5
           / \
          3   6
         / \
        2   4
       /
      1
k = 3  ->  Output: 3
```

**Constraints:**
- `1 <= k <= n <= 10^4` (n = number of nodes)
- `0 <= Node.val <= 10^4`
- Valid BST, all values unique.

**Follow-up:** If the BST is modified often (inserts/deletes) and k-th smallest is queried frequently, how would you optimize?

---

## Conversation Log

**Interviewer:** Presented problem; asked for start time and clarifying questions.

**Aayush:** 10:32 (start). What are the constraints?

**Interviewer:** Gave constraints; flagged the follow-up about a frequently-modified BST.

**Aayush:** The required element would be at the k-th position in the in-order traversal of the tree.

**Interviewer:** Correct — in-order of a BST is sorted. Asked whether the whole tree must be traversed or can it stop early.

**Aayush:** We can stop when we have found the k-th node.

**Interviewer:** Asked him to code it; provided boilerplate.

**Aayush:** (Recursive solution with `inorder(node, curPos&, ans&, k)` — increments curPos, sets ans at curPos==k and returns.)

**Interviewer:** Noted missing semicolon. Pressed: does the `return` actually stop traversal, or only exit the current frame? Trace the ancestor frames.

**Aayush:** No, traversal still completes for the parent's other subtrees.

**Interviewer:** (1) Is the returned answer still correct? (2) How to genuinely stop early?

**Aayush:** (1) Still correct — it holds the value of the node at the k-th position. (2) An iterative in-order where we break the loop when curPos == k.

**Interviewer:** Both correct. Asked him to write the iterative in-order with early break.

**Aayush:** (Iterative stack-based in-order with `break` at curPos==k — see solution.)

**Interviewer:** Traced on Example 2 → correct. Noted `if(!node)` is redundant but harmless. Asked for precise time and space (not just O(n)).

**Aayush:** Time O(k) since we only visit k elements; space O(H), H can be up to k in worst case.

**Interviewer:** Gave concrete case — left-skewed tree, n nodes, k=1. How many nodes pushed before the first pop? Is time really O(k)? Is H really up to k?

**Aayush:** It is O(n); similarly the stack can be O(n) for a left-skewed tree because all nodes are pushed before exploring.

**Interviewer:** Correct — time O(H + k), space O(H) with H up to n. Posed the follow-up: frequent modifications, frequent k-th queries — how to speed up?

**Aayush:** Knowing the left-subtree sizes helps determine the position of the current node, but this doesn't work for the right subtree of the root.

**Interviewer:** Nudged — it does work: if k == leftCount+1 answer is this node; if k <= leftCount go left; if k > leftCount+1 go right with k = k-(leftCount+1). Asked: query time? maintenance cost?

**Aayush:** Query is log n since each height level is visited once. No cost to maintain the counts since while reaching the affected node we can decrement the left-subtree counts of its ancestors.

**Interviewer:** Precision fix — query O(H) (log n only if balanced → order-statistic tree); maintenance is O(H) extra work on the path (free asymptotically, not literally free). Asked for end time.

**Aayush:** 11:16 (end).

---

## Solution
**Aayush's Final Solution (iterative, early-stop):**
```cpp
class Solution {
public:
    int kthSmallest(TreeNode* root, int k) {
        int ans = -1, curPos = 0;
        stack<TreeNode*> st;
        TreeNode* node = root;
        while (!st.empty() || node) {
            while (node) {
                st.push(node);
                node = node->left;
            }
            node = st.top();
            curPos++;
            st.pop();
            if (curPos == k) { ans = node->val; break; }
            node = node->right;
        }
        return ans;
    }
};
```

**Recursive version (correct, but does NOT truly early-stop):**
```cpp
void inorder(TreeNode* node, int& curPos, int& ans, int k) {
    if (!node) return;
    inorder(node->left, curPos, ans, k);
    curPos++;
    if (curPos == k) { ans = node->val; return; }   // only exits current frame
    inorder(node->right, curPos, ans, k);
}
```

**Follow-up (order-statistic tree — augment each node with left-subtree size):**
```
int rank-query(node, k):
    leftCount = size(node->left)
    if k == leftCount + 1: return node->val
    if k <= leftCount:     return query(node->left, k)
    else:                  return query(node->right, k - leftCount - 1)
```
Use a self-balancing BST (AVL / red-black) augmented with subtree sizes for guaranteed O(log n) query, insert, and delete.

**Time Complexity:** O(H + k) [stated O(k), corrected]
**Space Complexity:** O(H), H up to n [stated "H up to k", corrected]

---

## Feedback Given

**Overall:** Good round. Instantly recognized the BST in-order structure (no brute force), wrote a correct iterative early-stop traversal, and reasoned to the order-statistic-tree augmentation. Recurring soft spot: complexity precision — both time and space bounds were off until given a concrete skewed-tree case.

### Scoring

**Problem Understanding & Clarification — 4/5**
Asked for constraints; internalized 1-indexing and BST invariant without fuss.

**Approach & Thought Process — 4.5/5**
Strength today. Went straight for "k-th in in-order," exploiting structure. Reached for left-subtree counts on the follow-up; only needed a nudge to see the right-subtree re-indexing.

**Code Quality & Correctness — 4/5**
Iterative version correct and clean. Dings: (1) claimed the recursive version "stops" when it doesn't — saw it only when pushed; (2) minor missing semicolon. Answer stayed correct.

**Complexity Analysis — 3/5**
Said time O(k) — it's O(H + k) (dropped the descent). Said H "up to k" — it's up to n. Both from reasoning about the happy path and ignoring the skewed worst case. Corrected cleanly after the left-skewed example.

**Communication — 4/5**
Corrects gracefully, doesn't dig in. Reasoned the augmentation cost well; "no cost to maintain" was imprecise (O(H) on path, free asymptotically).

**Time Taken: 44 minutes**

### Top takeaway
Reason about the worst case explicitly before stating a complexity bound. Before saying "O(k)," ask "what's the most expensive shape this input can take?" — a skewed tree immediately exposes the hidden H term in both time and space.
