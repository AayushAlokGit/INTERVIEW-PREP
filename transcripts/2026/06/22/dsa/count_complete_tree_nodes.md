# DSA Round Transcript
**Date:** 2026-06-22
**Start Time:** 9:59
**End Time:** 11:14
**Duration:** 75 minutes
**Problem:** Count Complete Tree Nodes
**Topic:** Binary Trees, Binary Search, Bit Manipulation
**Difficulty:** Medium (optimal solution Hard-leaning)

---

## Problem Statement
Given the `root` of a **complete binary tree**, return the number of nodes in the tree. A complete binary tree has every level completely filled except possibly the last, with all last-level nodes as far left as possible. The expectation is to do better than O(n) (visiting every node).

**Example 1:** `root = [1,2,3,4,5,6]` → `6`
**Example 2:** `root = []` → `0`
**Example 3:** `root = [1]` → `1`

Constraints:
- Number of nodes in `[0, 5*10^4]`; `0 <= Node.val <= 5*10^4`.
- Tree guaranteed complete.

---

## Conversation Log

**Interviewer:** Presented the problem; hinted the expectation is to beat O(n). Asked for clarifying questions and initial approach.

**Aayush:** Noted start time 9:59. Asked for constraints.

**Interviewer:** Gave constraints; reiterated the implicit "better than O(n)" expectation.

**Aayush:** For a complete tree of height h, nodes = 1 + 2 + ... + 2^(h-1) + x, where x = nodes in the last level, x ≤ 2^h. Height findable in O(log n); counting x efficiently is the challenge.

**Interviewer:** Confirmed the decomposition (perfect tree above = 2^h − 1, plus x). Offered two directions: (1) binary search the last-level prefix; (2) compare all-left vs all-right height recursively. Asked for the cost of checking one position/subtree.

**Aayush:** Asked for examples.

**Interviewer:** Gave a concrete h=2 tree, walked through both directions, including the slot→binary-path mapping (bit 0 = left, 1 = right). Asked him to pick one and give the precise algorithm.

**Aayush:** Chose Direction 1. Slots 0..2^h−1; each slot's binary representation (h bits) encodes the root-to-node path (0→left, 1→right). Convert position to h bits, walk the path, check if the node exists; if not, the position is too far right, so reduce the search space to lower positions.

**Interviewer:** Locked in costs: one exists-check = O(h); binary search = O(h) checks; height = O(log n). Asked him to code it.

**Aayush:** Asked for boilerplate.

**Interviewer:** Provided the struct + class scaffold.

**Aayush:** Submitted first version (binary search + `traverse` bit-walk correct), but computed height via a full recursive `levels()` (max of left/right over all nodes).

**Interviewer:** Confirmed binary-search/traverse logic correct (returns 6 on example). Flagged `levels()` and asked: (1) its complexity and the overall complexity; (2) dry-run `root = nullptr` through `nodesCnt = (1 << treeHeight) - 1`.

**Aayush:** `levels()` is O(n); can be reduced to O(log n) by walking only left pointers. Null case needs handling.

**Interviewer:** Asked for both fixes plus precise overall complexity.

**Aayush:** Submitted corrected version: iterative `height()` walking left only, null guard `if(!root) return 0;`. Stated TC = O(h²), SC = O(h) for recursion stack.

**Interviewer:** Confirmed both correct and precise; traced null→0, single→1, example→6. Final optimization probe: can SC be O(1)?

**Aayush:** Make the traversal iterative — brings SC down, doesn't change TC.

**Interviewer:** Correct — O(1) space, O(log²n) time, fully optimal. Moved to feedback.

---

## Solution
**Aayush's Final Solution (corrected):**
```cpp
class Solution {
public:
  int countNodes(TreeNode* root) {
      if(!root) return 0;
      int treeHeight = height(root);
      int nodesCnt = (1<<treeHeight) - 1;
      int lo = 0, hi = (1<<treeHeight)-1;
      int ans = -1;
      while(lo <= hi) {
          int mid = lo + (hi-lo)/2;
          if(!exists(root,mid,treeHeight)) hi = mid-1;
          else { ans = mid; lo = mid+1; }
      }
      return nodesCnt + ans + 1;
  }
  int height(TreeNode* root) {
      int h = 0;
      while (root->left) { h++; root = root->left; }
      return h;
  }
  bool exists(TreeNode* root, int slot, int h) {
      return traverse(root, slot, h) != nullptr;
  }
  TreeNode* traverse(TreeNode* node, int mask, int h) {
      if(!node || h==0) return node;
      int msb = mask & (1 << (h - 1));
      if(msb) return traverse(node->right, mask, h-1);
      return traverse(node->left, mask, h-1);
  }
};
```

**Optimal refinement:** make `traverse` iterative to reach O(1) space (height is already iterative). Time stays O(log²n).

**Time Complexity:** O(h²) = O(log²n) — O(h) height + O(h) binary-search iterations × O(h) per exists-check.
**Space Complexity:** O(h) = O(log n) with recursive traverse; O(1) if made iterative.

---

## Feedback Given

**Strengths**
- Immediately produced the correct decomposition (perfect tree `2^h−1` + last-level prefix `x`) and recognized counting `x` was the crux.
- Picked the binary-search direction and nailed the slot→binary-path bit mapping; the `traverse` bit-walk was clean and correct on the first attempt.
- **Big improvement on complexity:** once the code was right, he stated TC = O(h²) and SC = O(h) precisely and correctly on the first try — directly addressing last round's weakest area.
- Diagnosed both flaws (O(n) height, null UB) correctly and immediately once pointed at them, and articulated the O(1)-space optimization without hesitation.

**Areas to improve**
1. **Reached for a generic O(n) height computation** (full recursive max-depth) inside an algorithm whose entire purpose was to beat O(n). The structure-exploiting move — height = walk-left-only — is exactly what the problem rewards. Recurring pattern: when a problem hinges on a structural property, make sure *every* sub-step exploits it, not just the headline step.
2. **Did not self-verify edge cases before declaring done.** `root = nullptr` hits `1 << -1` (undefined behavior); he caught it only when I asked him to dry-run it. A 10-second mental run on empty / single-node inputs would have surfaced both the null bug and the O(n) height cost.

**Scorecard**
| Category | Score |
|---|---|
| Problem Understanding & Clarification | 4.0 / 5 |
| Approach & Thought Process | 3.5 / 5 |
| Code Quality & Correctness | 3.5 / 5 |
| Complexity Analysis | 4.5 / 5 |
| Communication | 4.0 / 5 |

**Overall: 3.9 / 5** — Reached the fully optimal O(log²n) / O(1) solution with strong, precise complexity reasoning this time. Two habits to tighten: exploit the structure in *every* sub-step (don't drop to a generic O(n) helper), and dry-run empty/single-node inputs before declaring done.

**Time Taken: 75 minutes**
