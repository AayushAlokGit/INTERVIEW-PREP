# Serval Technical Round Transcript
**Date:** 2026-05-29
**Start Time:** 8:54 PM
**End Time:** 9:34 PM
**Duration:** 40 minutes

## Part 1 — Coding
**Problem:** Meeting Rooms II
**Topic:** Intervals / Heap / Sweep Line
**Difficulty:** Medium

### Problem Statement
Given intervals[i] = [start_i, end_i] for meetings, return the minimum number of conference rooms required.

Examples:
```
[[0,30],[5,10],[15,20]] → 2
[[7,10],[2,4]] → 1
```

Constraints: 1 <= n <= 10^4; 0 <= start < end <= 10^6; half-open intervals; not sorted.

### Conversation Log

**Interviewer:** Presented problem.

**Aayush:** "what are the constraints?"

**Interviewer:** Shared constraints + half-open semantics + not-sorted note proactively.

**Aayush:** "sort by start time and have a minheap of endTime. The minHeap size is the number of minimum conference rooms required."

**Interviewer:** Pushed for mechanics — when to pop, what to push, why max heap size = answer.

**Aayush:** "At each interval i, check if room with min end time is <= start[i]; if yes pop and push end[i], else just push. ans = max(ans, minH.size())."

**Interviewer:** Confirmed trace correctness, asked to code.

**Aayush:**
```cpp
for(int i=0;i<intervals.size();i++){
    if(minH.size() == 0){
        minH.push(intervals[i][1]);
        continue;
    }
    int minEndTime = minH.top();
    if(minEndTime <= intervals[i][0]) minH.pop();
    minH.push(intervals[i][1]);
    ans = max(ans, (int)minH.size());
}
```

**Interviewer:** "Trace `[[0,10]]`."

**Aayush:** "ans should be 1 as base case."

**Interviewer:** Pointed out his code outputs 0 due to `continue` skipping ans update.

**Aayush:** "update ans before continue."

**Interviewer:** Suggested cleaner: drop special case entirely; `if (!empty && top<=start) pop; push; update ans;`. Then asked for complexity + sweep line alternative.

**Aayush:** "O(n log n) time + O(n) space."

**Interviewer:** Hinted at sweep line.

**Aayush:** "not getting it"

**Interviewer:** Explained sweep line — sort starts and ends separately, two pointers, increment rooms on new start unless an end has freed one.

### Solution
**Aayush's Final Solution:** sort + min-heap of end times (with empty-heap special case fixed).

**Optimal:** sort + heap OR sweep line (sort starts and ends separately, two pointers).

**Time Complexity:** O(n log n)
**Space Complexity:** O(n)

---

## Part 2 — Schema Design
**Scenario:** Slack-style Messaging Workspace (workspaces, channels, messages, threads, reactions)

### Conversation Log

**Interviewer:** Presented scenario + 4 core queries.

**Aayush:** First sketch:
- Workspace(id, name)
- User(id, name)
- WorkspaceMembership(id, userId, workspaceId) — UNIQUE(userId, workspaceId)
- Channel(id, name, workspaceId)
- ChannelMembership(id, userId, channelId) — UNIQUE(userId, channelId)
- Message(id, channelId, msgText, userId, createdAt, parentMessageId)
- Reaction(id, emoji, messageId, userId)

**Interviewer:** Five pushbacks — drop surrogate IDs on M:N, add public/private to Channel, add Reaction constraints, threadRootMessageId vs parent, latest-50 index.

**Aayush:** Updated sketch:
- WorkspaceMembership(userId, workspaceId) — composite PK
- Channel(id, name, workspaceId, createdAt, public/private, createdBy)
- ChannelMembership(userId, channelId) — composite PK
- Message(id, channelId, msgText, userId, createdAt, threadRootMessageId)
  - Index: (createdAt, channelId) for latest-50
  - Index: threadRootMessageId for thread query
- Reaction(id, emoji, messageId, userId) — UNIQUE(messageId, userId)
  - Index: messageId

**Interviewer:** Three issues:
1. Reaction UNIQUE wrong direction — blocks user reacting with multiple emoji.
2. Channel index on "userId" meant ChannelMembership; noted composite PK gives "all channels for user U" for free.
3. Index column order — `(createdAt, channelId)` is wrong for `WHERE channel_id = X ORDER BY created_at DESC LIMIT 50`.

**Aayush:** "(channelId, createdAt) — first sort by channelId, in equality sort by createdAt." ✓

**Interviewer:** Asked offset vs cursor pagination.

**Aayush:** "Cursor — no need to skip rows."

**Interviewer:** Expanded: O(log n) per page vs O(n+offset); stability under inserts.

**Interviewer:** Soft delete — design and tradeoffs?

**Aayush:** "deleted_at column. No additional audit table, can recover unlike hard delete. Read areas need to handle the column."

**Interviewer:** Asked about index strategy after soft-delete — keep existing, expand to include deleted_at, or partial index?

**Aayush:** "partial index — most queries don't need deleted records."

**Interviewer:** Confirmed; explained partial index size/write benefits.

### Final Schema

| Table | Columns | Constraints / Indexes |
|---|---|---|
| Workspace | id (PK), name, created_at | |
| User | id (PK), name, email | UNIQUE(email) |
| WorkspaceMembership | user_id, workspace_id, joined_at | PK(user_id, workspace_id) |
| Channel | id (PK), name, workspace_id (FK), is_private (bool), created_at, created_by (FK→User) | INDEX(workspace_id) |
| ChannelMembership | user_id, channel_id, joined_at | PK(user_id, channel_id); INDEX(channel_id, user_id) for "members of C" |
| Message | id (PK), channel_id (FK), user_id (FK), text, created_at, thread_root_message_id (FK→Message, nullable), deleted_at (nullable) | INDEX(channel_id, created_at DESC) WHERE deleted_at IS NULL; INDEX(thread_root_message_id) |
| Reaction | id (PK), message_id (FK), user_id (FK), emoji, created_at | UNIQUE(message_id, user_id, emoji); INDEX(message_id) |

### Key Design Decisions
- Composite PKs on M:N tables — no surrogate id needed.
- Thread root pointer (not parent) — flat thread model = single index lookup for all replies.
- Partial index on (channel_id, created_at DESC) WHERE deleted_at IS NULL — keeps the hot path small.
- Cursor pagination for message lists — stable under writes, O(log n) per page.

---

## Feedback Given

**Coding scores:**
- Problem understanding & clarification: 2.5/5 — only asked for constraints when prompted (same as round 1).
- Approach & thought process: 4/5 — right family immediately; precise after one nudge.
- Code quality & correctness: 3/5 — empty-heap branch caused `[[0,10]]` to output 0.
- Complexity analysis: 4.5/5 — correct.
- Communication: 3/5 — didn't trace until pushed; didn't know sweep line.

**Coding avg: 3.4/5**

**Schema scores:**
- Entity identification: 4/5 — clean first pass.
- Table design: 3.5/5 — surrogate IDs initially; cleaned up cleanly.
- Indexing: 3/5 — wrong column order on composite; corrected on prompt.
- Normalization & tradeoffs: 4/5 — cursor vs offset and soft-delete tradeoffs both solid.
- Edge cases & data integrity: 2.5/5 — Reaction UNIQUE too restrictive (same UNIQUE-reasoning weak spot as round 1).
- Communication: 3.5/5 — terse but clear.

**Schema avg: 3.4/5**

**Recurring themes across rounds 1 + 2:**
1. UNIQUE constraint reasoning — round 1 too permissive, round 2 too restrictive. Walk through both "what does this allow" and "what does this block" before committing.
2. Clarifying questions — proactively ask about input ranges, semantics, edge inputs.
3. Edge case tracing — boundary cases (empty, single element) need to be checked before declaring done.
4. Index column order — equality first, range/order second.

**Time Taken: 40 minutes**
