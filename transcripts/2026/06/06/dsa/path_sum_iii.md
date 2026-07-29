# DSA Round Transcript
**Date:** 2026-06-06
**Start Time:** 20:00
**End Time:** 20:37
**Duration:** 37 minutes
**Problem:** Path Sum III
**Topic:** Binary Trees / Prefix Sum + Hashmap
**Difficulty:** Medium

---

## Problem Statement

Given the `root` of a binary tree and an integer `targetSum`, return the number of paths where the sum of the node values along the path equals `targetSum`. The path does not need to start at the root or end at a leaf, but it must go downwards (parent → child only).

**Example:**
```
        10
       /  \
      5    -3
     / \     \
    3   2     11
   / \   \
  3  -2   1

targetSum = 8  ->  returns 3
Paths: (5→3), (5→2→1), (10→-3→11)
```

**Constraints:**
- Number of nodes in range `[0, 1000]` (tree may be empty)
- `-10^9 <= node.val <= 10^9` (values can be negative)
- `-1000 <= targetSum <= 1000`
- Node values are not unique.

---

## Conversation Log

**Interviewer:** Presented the problem and asked for clarifying questions.

**Aayush:** "What are the constraints?"

**Interviewer:** Gave constraints — node count [0,1000] (empty possible), values can be negative (-1e9..1e9), targetSum in [-1000,1000].

**Aayush:** "Are node values unique?"

**Interviewer:** No — values are not unique (example has two 3s).

**Aayush:** "I'm thinking of an approach where we calculate the sum from root to the node moving downwards. At each node we check if curSum - target has been encountered at which nodes before. If this sum was encountered at an ancestor node x, then there's a path from x to current node with sum target. Keep a set to track the nodes currently in the stack and pop when done exploring subtrees."

**Interviewer:** Right instinct (prefix sums along the path). Probed: what exactly is stored in the set — nodes or sums? What is the lookup against? There's a subtlety.

**Aayush:** "We need two things: 1. Has this sum been seen before, and at which node — so a map of sum → node pointer. 2. Is a node an ancestor — so a set of node pointers."

**Interviewer:** Pushed on two points. (1) If you add prefix sum on the way down and remove when finishing the subtree, everything in the map is already an ancestor — is the separate set doing any work? (2) Given negatives + repeats, can two ancestors share the same prefix sum? What breaks about sum → single node?

**Aayush:** "Yes — we can simply store the count of prefix sums, increment when beginning to explore current node, decrement when done. Then checking curSum - target in the map gives non-zero only if some ancestor's root-prefix equals the required sum."

**Interviewer:** Correct on both — counts not pointers, and the ancestor set is redundant. Asked about root-starting paths (e.g. 10→-3→11): when at node 11, curSum=18, curSum-target=10 — how to ensure root-starting paths get counted?

**Aayush:** "Initialise the map with {0:1}."

**Interviewer:** Correct. Asked for full code.

**Aayush:** Requested C++ boilerplate.

**Interviewer:** Provided TreeNode struct + Solution skeleton with `unordered_map<long long,int>`; noted to think about why the key is `long long`.

**Aayush:** Submitted solution (see below).

**Interviewer:** Asked him to read it as a compiler — a couple of things won't compile.

**Aayush:** "return 0 → no need of 0, and target not targetSum."

**Interviewer:** Both correct. Flagged a stylistic subtlety with `operator[]` for later. Asked for time and space complexity.

**Aayush:** "TC is O(n) — each node visited at most once. SC is O(h) for the prefixCount map and recursion stack."

**Interviewer:** TC correct. Pushed on space: his code used `operator[]` for the lookup, which inserts permanent zero-keys on a miss — does the map really stay O(h)?

**Aayush:** "Right, so the map also becomes O(n)."

**Interviewer:** Correct. Asked how to fix it to keep the map O(h).

**Aayush:** "Use find and remove keys when backtracking."

**Interviewer:** Complete fix. Confirmed optimal solution reached. Asked for end time and delivered feedback.

---

## Solution

**Aayush's Final Solution (as submitted, pre-fix):**
```cpp
class Solution {
    void solve(TreeNode* node, long long curSum, int target,
               unordered_map<long long,int> &prefixCount, int &ans)
    {
        if(!node) return 0;                       // bug: return 0 in void
        curSum += node->val;
        prefixCount[curSum]++;
        ans += prefixCount[curSum - targetSum];   // bug: targetSum should be target; operator[] inserts keys
        solve(node->left, curSum, target, prefixCount, ans);
        solve(node->right, curSum, target, prefixCount, ans);
        prefixCount[curSum]--;
    }
public:
  int pathSum(TreeNode* root, int targetSum) {
      unordered_map<long long, int> prefixCount;
      int ans=0;
      prefixCount[0] = 1;
      solve(root, 0, targetSum, prefixCount, ans);
      return ans;
  }
};
```
Bugs found (when prompted): `return 0;` in a void function; `targetSum` used instead of the `target` parameter; `operator[]` lookup inflates map to O(n).

**Optimal Solution (clean, true O(n) time / O(h) space):**
```cpp
class Solution {
    int solve(TreeNode* node, long long curSum, int target,
              unordered_map<long long,int>& prefixCount) {
        if (!node) return 0;
        curSum += node->val;

        int count = 0;
        auto it = prefixCount.find(curSum - target);   // find(), not operator[]
        if (it != prefixCount.end()) count += it->second;

        prefixCount[curSum]++;
        count += solve(node->left,  curSum, target, prefixCount);
        count += solve(node->right, curSum, target, prefixCount);
        if (--prefixCount[curSum] == 0) prefixCount.erase(curSum);  // keep map O(h)

        return count;
    }
public:
    int pathSum(TreeNode* root, int targetSum) {
        unordered_map<long long,int> prefixCount;
        prefixCount[0] = 1;
        return solve(root, 0, targetSum, prefixCount);
    }
};
```

**Time Complexity:** O(n) — each node visited once, O(1) amortized work per node.
**Space Complexity:** O(h) with the `find()` + erase-on-backtrack fix (O(n) as originally written due to `operator[]` leak); plus O(h) recursion stack.

---

## Feedback Given

Strong round. Reached the optimal solution with clean reasoning.

**What went well:**
- Reached for the prefix-sum + hashmap pattern immediately, skipping the O(n^2) brute force.
- Self-corrected the two hardest conceptual points without being given the answer: sum→count (not sum→node) because negatives/repeats let ancestors share a prefix sum; and dropping the redundant ancestor set given increment-on-entry / decrement-on-exit.
- Good clarifying questions (constraints surfacing negatives; value uniqueness) before committing.
- Got `{0:1}` initialization for root-starting paths unprompted.

**What to tighten:**
- Two compile bugs shipped in first submission (`return 0;` in void; `targetSum` vs `target`). Found instantly when asked to "read as a compiler" — a 10-second self-review would catch these. Build the reflex: read every line as the compiler before declaring done.
- Space-complexity miss: claimed O(h) but actual code used `operator[]`, which inserts permanent zero-keys → O(n). Corrected well when prompted. Lesson: connect the data-structure API's side effects to the complexity claim — `operator[]` on a map is never a pure read.

**Scoring:**
| Criterion | Score | Notes |
|---|---|---|
| Problem understanding & clarification | 4.5 / 5 | Asked about constraints + uniqueness; both load-bearing. Could've named the empty-tree case explicitly. |
| Approach & thought process | 4.5 / 5 | Went straight to the optimal pattern; self-corrected the two key subtleties. |
| Code quality & correctness | 3 / 5 | Right structure, but two compile errors + the operator[] leak slipped through unverified. |
| Complexity analysis | 3.5 / 5 | TC nailed immediately; space needed a nudge to account for own code's behavior. |
| Communication | 4.5 / 5 | Concise, responsive to probes, no defensiveness, corrected cleanly. |

**Overall: 4 / 5** — senior-level pattern recognition; the gap is pre-submission verification discipline.

**Time Taken: 37 minutes**
