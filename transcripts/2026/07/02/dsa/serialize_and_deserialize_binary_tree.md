# DSA Round Transcript
**Date:** 2026-07-02
**Start Time:** 10:46
**End Time:** 12:10
**Duration:** 84 minutes
**Problem:** Serialize and Deserialize Binary Tree
**Topic:** Trees / BFS (Level-order) & DFS (Preorder)
**Difficulty:** Hard

---

## Problem Statement
Design an algorithm to serialize and deserialize a binary tree. There is no restriction
on how the serialization/deserialization algorithm should work — you just need to ensure
that a binary tree can be serialized to a string, and this string can be deserialized to
the original tree structure.

Implement two functions:
- `serialize(root)` → returns a string
- `deserialize(data)` → returns the reconstructed tree root

**Example:**
```
      1
     / \
    2   3
       / \
      4   5

Input:  root = [1, 2, 3, null, null, 4, 5]
Output: [1, 2, 3, null, null, 4, 5]   (round-trips back to the same tree)
```

**Constraints:**
- Number of nodes in range [0, 10^4].
- -1000 <= Node.val <= 1000 (values can be negative and multi-digit).
- Not a BST — no ordering guarantee.
- Duplicate values are allowed.
- Empty tree (root = null) must round-trip correctly.

---

## Conversation Log

**Interviewer:** (Presented the problem, asked for start time and clarifying questions.)

**Aayush:** 10:46 (start time)

**Interviewer:** Asked for clarifying questions and high-level approach.

**Aayush:** What are the constraints?

**Interviewer:** Gave constraints — [0, 10^4] nodes, -1000..1000 values, negatives/multi-digit, not a BST, general binary tree.

**Aayush:** Are there duplicates?

**Interviewer:** Yes, duplicates allowed — two different nodes can hold the same value.

**Aayush:** (Proposed) Use an array where node at index i has left child at 2*i+1 and right child at 2*i+2 (0-based). In serialize, build a hashmap index→value, then flatten to a string with '|' separators. In deserialize, split the string and use the child-index property to reconstruct.

**Interviewer:** Pushed on two points: (1) how does deserialize know which index each value belongs to; (2) for a 10,000-node left-skewed chain, what index does the deepest node get, and what does that mean for space?

**Aayush:** (1) For the example: index 0→1, 1→2, 2→3, 5→4, 6→5; missing indices are null nodes; minIndex=0, maxIndex found by iterating hashmap, missing indices represented by a reserved null character. (2) Space becomes exponential with the height of the tree; for a skewed tree the space needed is vastly more than the number of nodes.

**Interviewer:** Confirmed — deepest node lands at ~2^10000. Challenge: find an encoding whose serialized size is O(n) regardless of shape. Hint: record structure directly via a traversal with explicit null markers.

**Aayush:** Level-order (BFS) traversal, storing each node and its null children into the queue. For the example: level0 = 1; level1 = 2,3; level2 = null,null,4,5. String: `1|2|3|null|null|4|5`. Deserialize reads the string and applies 2*i+1 / 2*i+2 logic. Space is proportional to number of nodes since we only store nodes and their immediate null children.

**Interviewer:** Pointed out the contradiction — since children of null nodes are NOT emitted, the 2i+1/2i+2 index arithmetic breaks the moment an early null appears. Asked how deserialize can recover parent-child links from `1|2|3|null|null|4|5`. Hint: use a queue, same as building.

**Aayush:** Keep a queue of TreeNodes; root is always first, push it in. Maintain a pointer `cur` to the first value. In a while loop: pop the front node; its left child is at `cur`, right child at `cur+1`; if left not null push to queue, if right not null push to queue; increment `cur += 2`.

**Interviewer:** Confirmed the queue mechanism is correct. Asked him to write the full C++ code.

**Aayush:** Requested C++ boilerplate.

**Interviewer:** Provided boilerplate with TreeNode struct and Codec class.

**Aayush:** (Submitted full solution — see Solution section.)

**Interviewer:** Asked him to (1) dry-run deserialize on the example string and (2) reason about a potential out-of-bounds on the right-child read (only `cur < v.size()` checked at loop top).

**Aayush:** TC is O(n) where n is number of nodes and same for SC.

**Interviewer:** Redirected — he skipped both questions. Re-asked the out-of-bounds reasoning: is v.size() even or odd, and can the right read go past the end for a valid string?

**Aayush:** (1) Tokens: "1","2","3","nullptr","nullptr","4","5" — tree rebuilt. (2) Need to add a guard for right-child access.

**Interviewer:** Asked whether the guard is actually necessary — v.size() = 1 + 2*(non-null nodes); is that even or odd, and can the right read ever be out of bounds for validly serialized input?

**Aayush:** The number of nodes is always odd, so the guard is not really needed.

**Interviewer:** Confirmed. Asked for a precise space breakdown (BFS queue peak + string/vector size).

**Aayush:** The queue peaks at the maximum number of nodes in a level. Output string has 1 + 2*(non-null nodes), so proportional to n.

**Interviewer:** Confirmed O(n)/O(n), noted O(n) is a lower bound so the solution is optimal. Asked if there's a fundamentally different traversal some interviewers prefer.

**Aayush:** Not sure.

**Interviewer:** Nudged — you used BFS; what about preorder DFS with null markers?

**Aayush:** Preorder DFS with null markers. For deserialize, use the fact that in preorder the left child is always the next node. Start from i=0, push nodes to a stack building left-child bonds until a null is hit; when a new non-null node appears it's the right child of the stack top — pop and attach, and it also gets its own left child.

**Interviewer:** Acknowledged the iterative idea works but showed the clean recursive version (shared index passed by reference). Noted both BFS and DFS are O(n) time; DFS uses O(h) recursion stack vs O(n) queue. Moved to wrap-up and requested end time.

**Aayush:** 12:10 (end time)

---

## Solution

**Aayush's Final Solution (BFS):**
```cpp
struct TreeNode {
    int val;
    TreeNode* left;
    TreeNode* right;
    TreeNode(int x) : val(x), left(nullptr), right(nullptr) {}
};

class Codec {
public:
    string serialize(TreeNode* root) {
        string s;
        if(!root) return s;

        queue<TreeNode*> q;
        q.push(root);
        while(!q.empty())
        {
            TreeNode* node = q.front();
            q.pop();

            if(!node)
            {
                s += "nullptr|";
                continue;
            }
            s += to_string(node->val);
            s += "|";
            q.push(node->left);
            q.push(node->right);
        }
        // removing last '|'
        s = s.substr(0, s.size()-1);
        return s;
    }

    TreeNode* deserialize(string data) {
        if(data.size() == 0) return nullptr;

        vector<string> v;
        stringstream ss(data);
        string token;
        while (getline(ss, token, '|')) {
            v.push_back(token);
        }

        TreeNode* root = new TreeNode(stoi(v[0]));
        queue<TreeNode*> q;
        q.push(root);
        int cur = 1;
        while(!q.empty() && cur < v.size())
        {
            auto node = q.front();
            q.pop();

            TreeNode* lchild = nullptr;
            if(v[cur] != "nullptr") lchild = new TreeNode(stoi(v[cur]));
            cur++;

            TreeNode* rChild = nullptr;
            if(v[cur] != "nullptr") rChild = new TreeNode(stoi(v[cur]));
            cur++;

            node->left = lchild;
            node->right = rChild;

            if(lchild) q.push(lchild);
            if(rChild) q.push(rChild);
        }
        return root;
    }
};
```
*(Note: a stray debug `cout` loop was present in deserialize during the session; removed here.)*

**Optimal / Alternative Solution (DFS preorder, recursive):**
```cpp
class Codec {
public:
    void dfs(TreeNode* node, string& s) {
        if (!node) { s += "nullptr|"; return; }
        s += to_string(node->val) + "|";
        dfs(node->left, s);
        dfs(node->right, s);
    }
    string serialize(TreeNode* root) {
        string s;
        dfs(root, s);
        return s;
    }

    TreeNode* build(vector<string>& v, int& i) {
        if (v[i] == "nullptr") { i++; return nullptr; }
        TreeNode* node = new TreeNode(stoi(v[i++]));
        node->left  = build(v, i);
        node->right = build(v, i);
        return node;
    }
    TreeNode* deserialize(string data) {
        vector<string> v;
        stringstream ss(data);
        string token;
        while (getline(ss, token, '|')) v.push_back(token);
        int i = 0;
        return build(v, i);
    }
};
```

**Time Complexity:** O(n) — each node visited once in serialize and deserialize.
**Space Complexity:** O(n) — BFS queue peaks at the widest level (~n/2 for a complete tree); output string / token vector is O(n). (DFS variant: O(h) recursion stack, still O(n) worst case for a skewed tree.) O(n) is a lower bound, so the solution is asymptotically optimal.

---

## Feedback Given

### What went well
- Excellent clarifying questions — constraints, negatives/multi-digit, and duplicates (sharp; rules out value-keyed approaches). Step up from usual pattern of skipping range/semantics questions.
- Self-diagnosed the fatal flaw in the first idea — when pushed on the skewed tree, immediately saw the 2i+1/2i+2 scheme forces exponential (2^height) space and abandoned it cleanly.
- Clean, correct code on the first real attempt — BFS serialize/deserialize works, handles empty tree and single node, uses a good non-colliding sentinel, increments cur in pairs.
- Invariant reasoning landed — correctly derived v.size() = 1 + 2*(non-null nodes) is odd and concluded the guard is optional.

### What to work on
1. Skipped verification when asked — twice. When asked to dry-run deserialize and reason about the out-of-bounds read, answered "TC is O(n)" instead. When asked to verify, verify — don't substitute a different answer.
2. Defaults to "add a guard" instead of reasoning about whether it's needed. Know the invariant first, then decide.
3. "Not sure" on the alternative approach despite having all the pieces (preorder + null markers). Push one more step before conceding — got there with a nudge.
4. Pace — 84 minutes is well over a real 45-min slot; much of the overage was the verification back-and-forth.

### Scoring (out of 5)
| Category | Score | Notes |
|---|---|---|
| Problem Understanding & Clarification | 5/5 | Constraints, negatives, duplicates — thorough and proactive. |
| Approach & Thought Process | 4/5 | Great recovery from exponential-space idea to O(n) BFS; needed a nudge for the DFS alternative. |
| Code Quality & Correctness | 4.5/5 | Correct first pass, good edge handling. Stray debug cout in deserialize; minor. |
| Complexity Analysis | 4/5 | Correct O(n)/O(n) with good breakdown once prompted, but stated complexity to dodge a verification ask. |
| Communication | 3.5/5 | Clear when explaining, but dropped/substituted sub-questions when asked to trace — recurring pattern. |

**Overall: ~4.2/5** — strong session. Front-end (clarification, approach recovery) was genuinely excellent. The gap is entirely in verification discipline: when asked to trace or prove, do it directly instead of pivoting.

**Time Taken: 84 minutes**
