# Serval Technical Round Transcript
**Date:** 2026-05-31
**Start Time:** 11:01
**End Time:** 11:40
**Duration:** 39 minutes

## Part 1 — Coding

### Problem 1
**Problem:** Time-Based Key-Value Store
**Topic:** Hash Map + Binary Search
**Difficulty:** Medium

#### Problem Statement
Design a key-value store where each value is stored with a timestamp. Support:
- `set(key, value, timestamp)` — stores key→value at timestamp.
- `get(key, timestamp)` — returns the value set at the largest timestamp ≤ queried timestamp; `""` if none.

`set` calls for a given key arrive with strictly increasing, distinct timestamps.

Example:
```
set("foo","bar",1)
get("foo",1) -> "bar"
get("foo",3) -> "bar"
set("foo","baz",4)
get("foo",4) -> "baz"
get("foo",5) -> "baz"
```

#### Conversation Log
**Interviewer:** Present problem, invite clarifying questions.
**Aayush:** Can timestamps repeat for a key? Will timestamps for the same key always be increasing?
**Interviewer:** No repeats; yes, strictly increasing, so per-key vector is already sorted.
**Aayush:** `unordered_map<string, vector<pair<int,string>>>`. `set` pushes (timestamp,value) in O(1). `get` fetches the vector and runs `upper_bound` on query timestamp; required value is at idx-1. Asked: can query timestamp be less than least timestamp for a key?
**Interviewer:** Yes — return "" in that case (or if key absent).
**Aayush:** Wrote solution (below).
**Interviewer:** Traced all 4 examples → correct. Asked complexity, and what happens on `get` of a never-set key.
**Aayush:** set O(1), get O(log n), space O(#set ops). Initially skipped the missing-key question.
**Interviewer:** Pressed — `mp[key]` via operator[] default-inserts an empty vector for missing keys, silently growing the map.
**Aayush:** Fix: use `find`, return "" if not found.

#### Solution
**Aayush's Final Solution:**
```cpp
class KeyValueStore {
    unordered_map<string, vector<pair<int, string>>> mp;
public:
    void set(string key, string value, int timestamp) {
        mp[key].push_back({timestamp, value});
    }
    string get(string key, int timestamp) {
        auto &vec = mp[key];                 // bug: operator[] inserts on miss
        if (vec.empty()) return "";
        int upperBndIdx = upper_bound(
            vec.begin(), vec.end(), timestamp,
            [](int ts, const pair<int,string>& ele){ return ts < ele.first; }
        ) - vec.begin();
        int requiredIdx = upperBndIdx - 1;
        if (requiredIdx < 0) return "";
        return vec[requiredIdx].second;
    }
};
```
**Optimal Solution (fix for missing-key side-effect):**
```cpp
string get(string key, int timestamp) {
    auto it = mp.find(key);
    if (it == mp.end()) return "";
    auto &vec = it->second;
    int idx = upper_bound(vec.begin(), vec.end(), timestamp,
        [](int ts, const pair<int,string>& e){ return ts < e.first; }) - vec.begin();
    if (idx == 0) return "";
    return vec[idx-1].second;
}
```
**Time Complexity:** set O(1) amortized, get O(log n)
**Space Complexity:** O(number of set operations)

### Problem 2
**Problem:** Task Build Order (Course Schedule II variant)
**Topic:** Graph — Topological Sort (Kahn's)
**Difficulty:** Medium

#### Problem Statement
`n` tasks labeled `0..n-1`. `prerequisites[i] = [a,b]` means task `b` must complete before task `a`. Return any valid order to run all tasks, or `[]` if impossible (cycle).

Example 1: `numTasks=4, prerequisites=[[1,0],[2,0],[3,1],[3,2]]` → `[0,1,2,3]`.
Example 2: `numTasks=2, prerequisites=[[1,0],[0,1]]` → `[]`.

Constraints: `1 <= numTasks <= 2000`, no duplicate pairs, `0 <= a,b < numTasks`.

#### Conversation Log
**Interviewer:** Present problem.
**Aayush:** Asked for constraints.
**Interviewer:** Provided.
**Aayush:** Model as directed graph, edge b→a. Run Kahn's. If valid topo sort covers all nodes → DAG, schedulable; else cycle → [].
**Interviewer:** How do you detect the cycle?
**Aayush:** processedNodes < total tasks → cycle.
**Aayush:** Wrote solution with `addDependency` interface (below).
**Interviewer:** Stress-tested: `numTasks=5`, task 4 isolated. What does it output?
**Aayush:** Outputs 0,1,2,3 — recognized task 4 dropped; cycle check compares against indegree.size() which is also 4, so it passes incorrectly.
**Interviewer:** Root cause — anchored graph to edges, not the numTasks contract. Fix?
**Aayush:** Initialize indegree for all tasks 0..numTasks-1.
**Interviewer:** Correct; check becomes `order.size() != numTasks`. Also corrected SC to O(V+E).

#### Solution
**Aayush's Final Solution (as written, edge-anchored — buggy for isolated nodes):**
```cpp
class TaskScheduler {
    unordered_map<int, vector<int>> graph;
    unordered_map<int, int> indegree;
public:
    void addDependency(int prerequisite, int task) {
        graph[prerequisite].push_back(task);
        indegree[task]++;
        if (indegree.find(prerequisite) == indegree.end()) indegree[prerequisite] = 0;
    }
    vector<int> getExecutionOrder() {
        queue<int> q;
        for (auto &[task, degree] : indegree) if (degree == 0) q.push(task);
        vector<int> order;
        while (!q.empty()) {
            int cur = q.front(); q.pop();
            order.push_back(cur);
            for (int nxt : graph[cur]) if (--indegree[nxt] == 0) q.push(nxt);
        }
        if (order.size() != indegree.size()) return {};
        return order;
    }
};
```
**Optimal Solution (numTasks-anchored, handles isolated nodes):**
```cpp
vector<int> findOrder(int numTasks, vector<vector<int>>& prereqs) {
    vector<vector<int>> graph(numTasks);
    vector<int> indeg(numTasks, 0);
    for (auto& p : prereqs) { graph[p[1]].push_back(p[0]); indeg[p[0]]++; }
    queue<int> q;
    for (int i = 0; i < numTasks; i++) if (indeg[i] == 0) q.push(i);
    vector<int> order;
    while (!q.empty()) {
        int cur = q.front(); q.pop();
        order.push_back(cur);
        for (int nxt : graph[cur]) if (--indeg[nxt] == 0) q.push(nxt);
    }
    return order.size() == numTasks ? order : vector<int>{};
}
```
**Time Complexity:** O(V + E)
**Space Complexity:** O(V + E) (adjacency list holds all edges — Aayush initially said O(V))

---

## Part 2 — Schema Design
**Scenario:** URL Shortener with Analytics (Bitly-style)

### Conversation Log
**Interviewer:** Design schema. Start with entities and access patterns.
**Aayush:** Entities: User(id,email); ShortURL(userId, shortCode, createdAt, device, country); ShortUrlClick(shortCode, createdAt). Queries: (1) user's short urls, (2) long URL for a short code, (3) clicks for a URL.
**Interviewer:** device/country belong on the click, not the link. And ShortURL is missing the field needed to redirect.
**Aayush:** Corrected: ShortURL(userId, shortCode, createdAt, longUrl); ShortUrlClick(shortCode, createdAt, deviceClickedOn, clickLocation).
**Interviewer:** PKs, types, FKs?
**Aayush:** shortCode as PK ("monotonically increasing unique value"); shortCode & longUrl as strings. ShortUrlClick PK = (shortCode, createdAt) composite; reference shortUrl by shortCode.
**Interviewer:** Challenged — shortCode is random base62, not monotonic. Composite PK collides on two clicks at same timestamp. Concrete types?
**Aayush:** Add surrogate auto-increment id to ShortURL and to ShortUrlClick.
**Interviewer:** Then redirect still queries WHERE shortCode=?; what does shortCode need? FK by code or id? Concrete types?
**Aayush:** shortCode needs a UNIQUE index. (On FK) "can reference by id as well as shortCode."
**Interviewer:** Pushed for a decision + reasoning. Reference by id (narrower FK, immutable, smaller joins). Types?
**Aayush:** shortCode VARCHAR(10), longUrl TEXT, createdAt TIMESTAMP.
**Interviewer:** Prefer TIMESTAMPTZ (global clicks). Index for analytics? Scale?
**Aayush:** Composite index (short_url_id, createdAt). For scale: precomputation jobs storing precomputed aggregate numbers.
**Interviewer:** Correct — equality-then-range index; pre-aggregated rollup tables for dashboard.

### Final Schema
**User**
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK (auto) | |
| email | VARCHAR(255) UNIQUE | |

**ShortURL**
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK (auto) | surrogate PK |
| user_id | BIGINT FK→User(id) | |
| short_code | VARCHAR(10) UNIQUE | unique index serves redirect lookup |
| long_url | TEXT | |
| created_at | TIMESTAMPTZ | |

**ShortUrlClick**
| Column | Type | Notes |
|---|---|---|
| id | BIGINT PK (auto) | avoids timestamp-collision PK |
| short_url_id | BIGINT FK→ShortURL(id) | reference by id, not code |
| created_at | TIMESTAMPTZ | |
| device | VARCHAR | |
| country | VARCHAR | |

Indexes: UNIQUE(short_code) on ShortURL; composite (short_url_id, created_at) on ShortUrlClick.

Rollup (for scale): `daily_click_counts(short_url_id, date, count)` populated by batch/streaming job.

### Key Design Decisions
- Surrogate `id` PKs on ShortURL and ShortUrlClick: avoids composite-PK timestamp collisions on high-volume clicks.
- UNIQUE index on `short_code`: fast redirect lookup + enforces no duplicate codes (constraint provides the index for free).
- FK by surrogate `id` not `short_code`: narrower/immutable FK, smaller joins.
- Composite index `(short_url_id, created_at)`: equality-then-range, serves both "all clicks for link" and time-windowed queries.
- TIMESTAMPTZ over TIMESTAMP: global click sources, store UTC.
- Pre-aggregated rollup tables for dashboard aggregates to avoid scanning raw click rows.

---

## Feedback Given

### Part 1 — Coding
Went well: good clarifying questions on both; correct, efficient approaches reached quickly (hash map + upper_bound with correct comparator; Kahn's with right cycle signal); clean code.
Work on: doesn't self-catch edge cases — two real bugs surfaced only on probing (operator[] insert side-effect; edge-anchored graph dropping isolated tasks while the cycle check fails to catch it). Complexity precision — said graph space O(V) instead of O(V+E).

### Part 2 — Schema Design
Went well: led with access patterns; strong indexing (UNIQUE on code, correct composite index order); reached pre-aggregation for scale.
Work on: first-pass entities had errors (misplaced device/country, missing longUrl); monotonic shortCode misconception; composite PK collision missed until prompted; indecisiveness ("either works" on FK) and terse, minimal-elaboration answers.

### Scoring
Coding (combined):
- Problem understanding & clarification: 4 — good clarifying Qs on both
- Approach & thought process: 4 — correct, efficient approaches reached quickly
- Code quality & correctness: 3 — two real bugs surfaced only on probing
- Complexity analysis: 4 — mostly right; missed E term in graph space
- Communication: 4 — clear reasoning, doesn't volunteer edge cases unprompted

Schema Design:
- Entity identification & requirements clarification: 3 — led with access patterns but misplaced click attrs, missed longUrl
- Table design (keys, types, relationships): 3 — monotonic misconception, PK collision, FK indecision (all corrected with prompts)
- Indexing & query optimization: 4 — UNIQUE on code; correct composite index order
- Normalization & tradeoffs: 4 — reached pre-aggregation rollups for scale
- Edge cases & data integrity: 3 — missed timestamp-collision race and timezone until prompted
- Communication: 3 — terse; "either works" hedging; minimal unprompted elaboration

Overall: Strong algorithmic foundation; approaches correct and fast. Gap is verification rigor — catch own edge cases / contract violations before being prompted — and on schema, commit to decisions with reasoning instead of hedging.

**Time Taken: 39 minutes**
