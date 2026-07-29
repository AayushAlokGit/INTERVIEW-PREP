# Serval Technical Round Transcript
**Date:** 2026-06-03
**Start Time:** 11:23
**End Time:** 11:55
**Duration:** 32 minutes

## Part 1 — Coding

### Problem 1
**Problem:** Trending Searches (Top K Frequent Elements)
**Topic:** Hash map + Heap
**Difficulty:** Medium

#### Problem Statement
Given a list of search query strings logged over the last hour and an integer `k`, return the `k` most frequently searched queries.

Example:
```
queries = ["pizza", "sushi", "pizza", "tacos", "sushi", "pizza"]
k = 2
→ ["pizza", "sushi"]
```

Constraints: n up to ~10^6; distinct queries (m) up to n; 1 ≤ k ≤ distinct count; lowercase strings ≤ 50 chars; no ties.

#### Conversation Log
**Interviewer:** Presented problem; asked for clarifying questions.
**Aayush:** Asked for constraints.
**Interviewer:** Provided constraints; asked for approach.
**Aayush:** Frequency map via iteration O(n), then sort by frequency O(n log n), return top k. Proactively handled ties (lexicographical) and output order (sorted by frequency).
**Interviewer:** Pushed on complexity precision (sort is over distinct m, not n) and whether full sort is needed for small k.
**Aayush:** Corrected to m distinct; proposed min-heap of size k.
**Interviewer:** Asked for full complexity in n, m, k.
**Aayush:** O(n) for frequencies + O(m log k) for heap. Correct.
**Interviewer:** Asked to code.
**Aayush:** Wrote correct C++ solution using ordered `map` + size-k min-heap.
**Interviewer:** Noted `map` makes counting O(n log m) not O(n); asked about output order and k>m.
**Aayush:** Use unordered_map; store heap items in vector and reverse for descending order.

#### Solution
**Aayush's Final Solution:**
```cpp
map<string,int> mp; // should be unordered_map for O(n)
priority_queue<pair<int,string>, vector<pair<int,string>>, greater<pair<int,string>>> minH;
for(string s:queries) mp[s]++;
for(auto [str,cnt]:mp){
    minH.push({cnt,str});
    while(!minH.empty() && minH.size() > k) minH.pop();
}
// pop into vector, reverse for most-frequent-first
```
**Optimal Solution (if different):** Same approach; use `unordered_map` for O(n) counting. Alternative: bucket sort by frequency for O(n + m) overall.
**Time Complexity:** O(n + m log k)
**Space Complexity:** O(m)

### Problem 2
**Problem:** Service Outage Propagation (Rotting Oranges variant)
**Topic:** Graph / Multi-source BFS
**Difficulty:** Medium

#### Problem Statement
`m x n` grid of cells: 0 = empty, 1 = healthy server, 2 = failed server. Each minute, healthy servers 4-directionally adjacent to a failed server also fail. Return minimum minutes until no healthy server remains, or -1 if impossible.

Example:
```
grid = [[2,1,1],[1,1,0],[0,1,1]] → 4
```

Constraints: m,n up to ~500; values in {0,1,2}; consider grids with no failures or no healthy servers.

#### Conversation Log
**Interviewer:** Presented problem; asked for clarifying questions.
**Aayush:** Asked for constraints.
**Interviewer:** Provided constraints (including empty-case edges).
**Aayush:** Multi-source BFS, level count = minutes; leftover healthy ⇒ -1. Gave O(mn) time (V + up to 4*mn E) and O(mn) space.
**Interviewer:** Asked to code, warned about off-by-one level count and the two empty cases.
**Aayush:** Wrote correct C++ solution with `healthy == 0 → 0` early return, level-by-level BFS, `-1` fallback. Returns 4 on example.
**Interviewer:** Probed: if `&& healthy > 0` were dropped from while condition, would minutes change?
**Aayush:** "Extra minute" — correct; the trailing level over-counts by one.

#### Solution
**Aayush's Final Solution:**
```cpp
int minMinutesToFailAllServers(vector<vector<int>>& grid){
    int m=grid.size(), n=grid[0].size();
    queue<pair<int,int>> q; int healthy=0;
    for(int i=0;i<m;i++) for(int j=0;j<n;j++){
        if(grid[i][j]==2) q.push({i,j});
        else if(grid[i][j]==1) healthy++;
    }
    if(healthy==0) return 0;
    int minutes=0, dirs[4][2]={{1,0},{-1,0},{0,1},{0,-1}};
    while(!q.empty() && healthy>0){
        int sz=q.size();
        for(int i=0;i<sz;i++){
            auto [r,c]=q.front(); q.pop();
            for(auto&d:dirs){
                int nr=r+d[0], nc=c+d[1];
                if(nr>=0&&nr<m&&nc>=0&&nc<n&&grid[nr][nc]==1){
                    grid[nr][nc]=2; healthy--; q.push({nr,nc});
                }
            }
        }
        minutes++;
    }
    return (healthy==0)? minutes : -1;
}
```
**Optimal Solution (if different):** Same. Nit: guard `grid.empty()` before `grid[0].size()`.
**Time Complexity:** O(m*n)
**Space Complexity:** O(m*n)

---

## Part 2 — Schema Design
**Scenario:** Social Media Feed (posts, likes, comments, follows, home feed)

### Conversation Log
**Interviewer:** Asked for core entities AND access patterns — access patterns first.
**Aayush:** Jumped to tables (Post, User, FollowRelationship, PostMedia); omitted likes/comments.
**Interviewer:** Pulled back — enumerate access patterns first; add the missing like/comment entities.
**Aayush:** Access patterns: get feed, see followed users, get likes/comments for a post, get posts liked by a user. Added PostLike{postId+userId} and PostComment{postId+userId}.
**Interviewer:** Challenged the two key choices — comment composite key (comment twice?) and follow surrogate id.
**Aayush:** Comment composite blocks a second comment → use surrogate commentId PK. Follow: drop surrogate, composite (followerId, targetUserId) sufficient. Proposed indexes.
**Interviewer:** Pressure-tested indexes — PostLike PK already covers postId-prefixed queries; PostComment index has userId in the middle.
**Aayush:** Correct that posts-liked-by-user needs userId-first index. Then: comment index should be (postId, createdAt), no userId.
**Interviewer:** Probed home feed query at scale, and like-count strategy.
**Aayush:** Feed = fetch followed users then their posts; at scale joins introduce latency → precompute the feed. Like count: don't COUNT(*) per render → denormalized likeCount on Post, updated per like; at high traffic batch increments and accept eventual-consistency delay.

### Final Schema
**User** ( userId PK, name )
**Post** ( postId PK, authorId FK→User, text, createdAt )
**PostMedia** ( id PK, postId FK→Post, s3Url, status[pending/uploaded] )
**Follow** ( followerId FK→User, targetUserId FK→User, createdAt, **PK (followerId, targetUserId)** )
**PostLike** ( postId FK→Post, userId FK→User, createdAt, **PK (postId, userId)** )
**PostComment** ( **commentId PK**, postId FK→Post, userId FK→User, text, createdAt )

Denormalized: `Post.likeCount` (batched updates).

Indexes:
- Post: (authorId, createdAt) — a user's posts newest-first
- PostLike: PK (postId, userId) covers likes-for-post & did-user-like; add (userId, postId) for posts-liked-by-user
- PostComment: (postId, createdAt) — comments for a post newest-first

### Key Design Decisions
- **PostComment surrogate PK:** a user can comment many times on a post; composite (postId,userId) wrongly blocks repeats.
- **Follow composite PK (followerId, targetUserId):** enforces can't-follow-twice for free; no surrogate needed.
- **PostLike composite PK (postId, userId):** enforces one-like-per-user; PK index serves postId-prefixed reads for free.
- **Index column order:** equality column first, range/sort column last — (postId, createdAt) for comment feed.
- **Denormalized likeCount + precomputed feed:** trade write amplification / eventual consistency for fast reads; hybrid push/pull to handle celebrity fan-out.

---

## Feedback Given

**Time Taken: 32 minutes**

Overall: a strong round. Both coding problems solved correctly with optimal complexity, with clean self-correction under pushback. Schema round was good once warmed up, with a recurring process gap (jumping to tables) and a few key/index choices needing prompting.

### Coding (both problems combined)
- Problem understanding & clarification — **5/5**: Asked constraints on both; proactively nailed ties + output order on P1.
- Approach & thought process — **5/5**: Reached size-k min-heap and multi-source BFS instinctively.
- Code quality & correctness — **5/5**: Both compiled and correct; `healthy > 0` BFS guard a precise off-by-one fix.
- Complexity analysis — **4/5**: Strong O(n + m log k) and V+E breakdown, but stated O(n) counting while using ordered `map`; mismatch surfaced only on prompt.
- Communication — **4/5**: Clear and self-corrects, but occasionally terse ("extra minute" vs a one-line trace).

### Schema Design
- Entity identification & requirements — **3/5**: Jumped to tables despite explicit ask for access patterns first; missed likes/comments initially.
- Table design (keys, types, relationships) — **4/5**: Fixed comment surrogate PK + follow composite PK correctly, but only after probing.
- Indexing & query optimization — **4/5**: Landed (postId, createdAt) and redundant-index point, but didn't note PK-free-index unprompted; first index mis-ordered userId.
- Normalization & tradeoffs — **5/5**: Best part — denormalized likeCount with batched writes, named the read query and consistency tradeoff.
- Edge cases & data integrity — **4/5**: Caught comment-twice and follow-uniqueness once prompted.
- Communication — **4/5**: Terse early, sharp closing scale reasoning (precompute feed, hybrid).

### Top 3 things to work on
1. In schema rounds, enumerate access patterns BEFORE drawing tables.
2. Keep stated complexity in sync with the actual data structure (map vs unordered_map).
3. Remember PK/UNIQUE constraints give you an index for free — don't add redundant secondary indexes.
