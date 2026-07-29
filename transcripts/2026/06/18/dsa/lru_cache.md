# DSA Round Transcript
**Date:** 2026-06-18
**Start Time:** 17:25
**End Time:** 18:23
**Duration:** 58 minutes
**Problem:** LRU Cache
**Topic:** Design / Hashmap + Doubly Linked List
**Difficulty:** Medium

---

## Problem Statement
Design an LRU (Least Recently Used) cache. Implement `LRUCache`:
- `LRUCache(int capacity)` — initialize with positive capacity.
- `int get(int key)` — return value if present else -1; marks key most-recently-used.
- `void put(int key, int value)` — insert/update; if size would exceed capacity, evict the least recently used key.
Both `get` and `put` must be O(1) average.

**Example:** capacity=2; put(1,1); put(2,2); get(1)->1; put(3,3) evicts 2; get(2)->-1; put(4,4) evicts 1; get(3)->3; get(4)->4.

**Constraints:** 1 <= capacity <= 3000; 0 <= key <= 1e4; 0 <= value <= 1e5; up to 2e5 calls; O(1) average per op.

---

## Conversation Log

**Interviewer:** Presented problem, asked for clarifying questions.
**Aayush:** "what are the constraints?"
**Interviewer:** Gave constraints.
**Aayush:** Proposed hashmap for key-value + doubly linked list for recency (front = most recent, back = LRU); needs pointer to a key's node to splice in O(1); on get/put move node to front; on full, evict back node; maintain a second map key -> node pointer.
**Interviewer:** Confirmed correct canonical design; noted the two maps can be collapsed into one key->node. Asked him to code carefully (pointer updates / edge cases).
**Aayush:** Asked for boilerplate.
**Interviewer:** Gave a minimal skeleton (Node struct, members, signatures); left DLL setup and logic to him.
**Aayush:** Wrote full LRUCache (no sentinels): head/tail pointers, updateRecency() splice helper, get(), put() with eviction. Tested only the given capacity=2 example.
**Interviewer:** Noted it passes capacity=2; asked him to trace the hostile input capacity=1, specifically the eviction block lines (tail=tail->prev; keyToNode.erase(tail->next->key); tail->next=nullptr).
**Aayush:** Posted a "fix": guarded with `if(tail->next)` and added `if(capacity==1) head=nullptr`. Explained the single-node case made tail null -> segfault on next.
**Interviewer:** Pointed out `if(tail->next)` STILL dereferences a null tail; and separately, for capacity=1 the erase lives inside that block so the evicted key never leaves keyToNode.
**Aayush:** "should check tail not tail->next typo".
**Interviewer:** `if(tail)` stops the crash but now for capacity=1 the erase block is skipped entirely -> key 1 stays in map -> get(1) returns stale node instead of -1, operating on a detached node. Root cause: reaches evicted node via tail->next after moving tail. Asked how to restructure (what to grab before reassigning tail).
**Aayush:** "save an oldTail, delete it, then reassign current tail."
**Interviewer:** Correct. Asked for the corrected eviction block.
**Aayush:** Wrote: capture oldTail=tail; tail=tail->prev; if(oldTail) erase oldTail->key; if(tail) tail->next=nullptr; if(capacity==1) head=nullptr; size--.
**Interviewer:** Traced capacity=1 -> now correct; multi-node still correct. Noted minor leak (no `delete oldTail`). Asked complexity and a robustness/optimization question.
**Aayush:** "O(1) for get and put, SC is O(capacity)."
**Interviewer:** Correct. Asked the robustness technique to eliminate all the null-check edge cases.
**Aayush:** "a dummy head and dummy tail with -1 key and -1 value."
**Interviewer:** Confirmed sentinel nodes; explained remove/insert become uniform and the single-node/empty/capacity-1 special cases vanish. Wrapped up.

---

## Solution

**Aayush's Final Solution (corrected, no sentinels):**
```cpp
struct Node {
  int key, value;
  Node* prev; Node* next;
  Node(int k, int v) : key(k), value(v), prev(nullptr), next(nullptr) {}
};

class LRUCache {
  int capacity, size;
  unordered_map<int, Node*> keyToNode;
  Node* head; Node* tail;

  Node* updateRecency(int key) {
      Node* curNode = keyToNode[key];
      if(head->key == curNode->key) return curNode;
      Node* prev = curNode->prev; Node* nxt = curNode->next;
      if(prev) prev->next = nxt;
      if(nxt)  nxt->prev = prev;
      if(curNode->key == tail->key) tail = prev;
      curNode->next = head;
      if(head) head->prev = curNode;
      head = curNode; head->prev = nullptr;
      return curNode;
  }
public:
  LRUCache(int capacity) : capacity(capacity), size(0), head(nullptr), tail(nullptr) {}

  int get(int key) {
      if(keyToNode.find(key) == keyToNode.end()) return -1;
      return updateRecency(key)->value;
  }

  void put(int key, int value) {
      if(keyToNode.find(key) == keyToNode.end()) {
          if(size == capacity) {                 // EVICT (the part that needed 3 fixes)
              Node* oldTail = tail;
              tail = tail->prev;
              if(oldTail) keyToNode.erase(oldTail->key);
              if(tail) tail->next = nullptr;
              if(capacity == 1) head = nullptr;
              size--;
              // (should also: delete oldTail;)
          }
          Node* newNode = new Node(key, value);
          keyToNode[key] = newNode;
          newNode->next = head;
          if(head) head->prev = newNode;
          head = newNode; head->prev = nullptr;
          if(size == 0) tail = newNode;
          size++;
      }
      Node* curNode = updateRecency(key);
      curNode->value = value;
  }
};
```

**Optimal / Recommended Solution (sentinel nodes — eliminates the bug class):**
```cpp
// dummyHead <-> dummyTail always present.
// remove(node): node->prev->next = node->next; node->next->prev = node->prev;
// insertFront(node): insertAfter(dummyHead, node);
// evict: remove(dummyTail->prev), erase its key.
// No null checks; capacity=1 / empty / head / tail are no longer special cases.
```

**Time Complexity:** O(1) average for get and put.
**Space Complexity:** O(capacity).

---

## Feedback Given

**Scoring**

1. **Problem Understanding & Clarification — 4/5** — Asked for constraints; did not probe boundary cases (capacity=1, update-existing, get-missing) that ended up breaking the code.

2. **Approach & Thought Process — 5/5** — Immediate, correct canonical design (hashmap + DLL + key->node) with correct O(1) reasoning.

3. **Code Quality & Correctness — 2.5/5** — First version passed capacity=2 example but segfaulted on capacity=1 (null deref in eviction). Took THREE iterations to fix: fix#1 `if(tail->next)` still derefs null tail; fix#2 `if(tail)` skips erasing evicted key -> stale get(1); fix#3 (save oldTail) correct. Did not default to sentinel nodes (the robust standard). Minor leak (no delete oldTail).

4. **Complexity Analysis — 5/5** — Immediate and correct (O(1) ops, O(capacity) space).

5. **Communication — 3.5/5** — Engaged, diagnosed each issue correctly once exact lines were pointed at, but needed pointing three times rather than reasoning about the eviction invariant for any list size.

**Overall:** Design and complexity are senior-level and instant. Execution at boundaries remains the bottleneck — ~3 debug cycles on a fully-understood problem. Two habits would have collapsed the round: (1) default to sentinel-node DLL construction so the null-check bug class can't exist; (2) run capacity=1 / empty / single-element BEFORE declaring done. Fourth consecutive round with a boundary bug masked by a friendly example.

**Time Taken: 58 minutes**
