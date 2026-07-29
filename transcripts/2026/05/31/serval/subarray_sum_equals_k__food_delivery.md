# Serval Technical Round Transcript
**Date:** 2026-05-31
**Start Time:** 8:57
**End Time:** 9:39
**Duration:** 42 minutes

## Part 1 — Coding
**Problem:** Subarray Sum Equals K
**Topic:** Arrays / Hash Map / Prefix Sum
**Difficulty:** Medium

### Problem Statement
Given an array of integers `nums` and an integer `k`, return the total number of contiguous subarrays whose elements sum to exactly `k`. A subarray is a contiguous, non-empty slice.

Constraints: `1 <= nums.length <= 2*10^4`, `-1000 <= nums[i] <= 1000` (negatives and zeros allowed), `-10^7 <= k <= 10^7` (any sign).

Example: `nums = [1,1,1], k = 2` → Output `2` (the two overlapping `[1,1]` subarrays).

### Conversation Log
**Interviewer:** Presented problem; invited clarifying questions.
**Aayush:** Asked for constraints, whether negatives are allowed in `nums`, whether `k` can be negative.
**Interviewer:** Gave constraints; confirmed negatives and zeros allowed, `k` any sign.
**Aayush:** Proposed prefix sum + map of `prefixSum -> indices`; at index `i`, if `prefix[i]-k` is in the map, add the count of those indices; append `i` to `map[prefix[i]]`.
**Interviewer:** Pushed on (1) whether indices are needed vs. just counts, (2) trace `[3], k=3`.
**Aayush:** Recognized only counts are needed; seed map with `{0:1}`. Traced `[3],k=3`: at i=0 prefix=3, lookup `3-3=0` found count 1, ans=1. Correct.
**Interviewer:** Asked for full code.
**Aayush:** Wrote correct C++ using `long long`, `std::map`, running prefix array. Stated O(n) time, O(n) space.
**Interviewer:** Challenged complexity claim (`std::map` cost) and the full prefix array.
**Aayush:** Recognized `std::map` is O(log n) → use `unordered_map` for O(1) amortized; replace prefix array with a running sum.
**Interviewer:** Edge case `[0,0,0], k=0`.
**Aayush:** Answered 6 (correct — n(n+1)/2).

### Solution
**Aayush's Final Solution (as written, with noted improvements):**
```cpp
#include <bits/stdc++.h>
using namespace std;

int main() {
    vector<int> nums {1,1,1};
    int k = 2;
    int n = nums.size();
    vector<long long> prefix(n);
    prefix[0] = nums[0];
    for (int i = 1; i < n; i++) prefix[i] = prefix[i-1] + nums[i];

    map<long long,long long> prefixSumCnt;   // -> unordered_map for O(1)
    prefixSumCnt[0] = 1;
    long long ans = 0;
    for (int i = 0; i < n; i++) {
        ans += prefixSumCnt[ prefix[i] - k ];
        prefixSumCnt[prefix[i]]++;
    }
    cout << "ans is " << ans;
    return 0;
}
```
**Optimal Solution (running sum + unordered_map):**
```cpp
long long subarraySum(vector<int>& nums, int k) {
    unordered_map<long long,long long> cnt;
    cnt[0] = 1;
    long long sum = 0, ans = 0;
    for (int x : nums) {
        sum += x;
        auto it = cnt.find(sum - k);
        if (it != cnt.end()) ans += it->second;
        cnt[sum]++;
    }
    return ans;
}
```
**Time Complexity:** O(n) with `unordered_map` (he initially said O(n) but used `std::map` → actually O(n log n); corrected when prompted).
**Space Complexity:** O(n) for the hashmap; running sum removes the need for the prefix array.

---

## Part 2 — Schema Design
**Scenario:** Food delivery platform (DoorDash / Uber Eats) — customers browse restaurants, place orders, drivers deliver.

### Conversation Log
**Interviewer:** Asked for core entities and key access patterns before any columns.
**Aayush:** Entities: Customer, Restaurant, Order, Item, Driver. Patterns: customer queries restaurants by category/nearby; customer queries a restaurant's menu; drivers query orders to deliver.
**Interviewer:** Pushed for more access patterns (customer order history/in-flight, restaurant dashboard, in-flight tracking); asked for the `Order` table in detail.
**Aayush:** `Order(id, userId, restaurantId, status, driverId, createdAt)` with composite indexes `(userId, createdAt)`, `(restaurantId, createdAt)`, `(status, createdAt)`.
**Interviewer:** Pointed out no link to items; asked how items are captured and the price-history question (old order with since-changed price).
**Aayush:** `OrderItem(id, orderId, itemId, price = snapshot at buy time)`.
**Interviewer:** Asked to name the principle, add quantity, reconsider the PK.
**Aayush:** Denormalization; add `quantity`; `(orderId, itemId)` composite PK.
**Interviewer:** Asked for the denormalization cost, composite-PK break case, and the driver geo query.
**Aayush:** Denormalized price may go stale but that's intended (historical correctness). Agreed `(status, createdAt)` not ideal for drivers. For "near them": geographic distance between driver and pickup; use PostgreSQL + PostGIS + geospatial indices.
**Interviewer:** Three follow-ups: which table holds coords, index type, status filter optimization.
**Aayush:** Restaurant table for pickup; guessed quadtree for index (unsure); did not answer the status-filter part initially.
**Interviewer:** Revealed GiST/R-tree + `ST_DWithin`; introduced partial index.
**Aayush:** Named "partial index"; asked the sharp question of *which column* the partial index covers.
**Interviewer:** Explained: denormalize pickup coords onto Order, then `GIST(pickup_location) WHERE status='ready' AND driver_id IS NULL`.

### Final Schema
```
Customer(id PK, name, email, ...)
Restaurant(id PK, name, category, location GEOGRAPHY, ...)
Item(id PK, restaurantId FK, name, price, ...)
Driver(id PK, name, location GEOGRAPHY, ...)

Order(
  id PK,
  userId FK -> Customer,
  restaurantId FK -> Restaurant,
  driverId FK -> Driver (nullable),
  status,                         -- enum: placed/ready/in_progress/delivered/...
  pickup_location GEOGRAPHY,      -- denormalized from Restaurant (for geo + history)
  createdAt
)
  INDEX (userId, createdAt)       -- customer order history
  INDEX (restaurantId, createdAt) -- restaurant dashboard
  PARTIAL INDEX GIST(pickup_location) WHERE status='ready' AND driver_id IS NULL  -- driver dispatch

OrderItem(
  orderId FK -> Order,
  itemId  FK -> Item,
  quantity,
  price,                          -- snapshot at purchase time (denormalized)
  PRIMARY KEY (orderId, itemId)
)
```

### Key Design Decisions
- **Composite indexes `(owner, createdAt)`**: serve "list my orders newest-first" for customer and restaurant.
- **Price snapshot on `OrderItem`**: denormalization for historical correctness — old orders show the price paid, not the current menu price; staleness is intended/immutable.
- **Composite PK `(orderId, itemId)`**: natural key with `quantity`; breaks only when the same item appears twice with different customizations (then add customization to key or use a surrogate id).
- **Geo dispatch**: pickup coordinates denormalized onto `Order`; PostGIS GiST index + `ST_DWithin`.
- **Partial index**: index only `status='ready' AND driver_id IS NULL` rows — tiny hot slice vs. millions of historical orders, fast and memory-cheap.

---

## Feedback Given

### Part 1 — Coding
Excellent. Clarified value ranges (negatives/zeros/k-sign) immediately — exactly what avoids a broken sliding-window. Reached optimal prefix-sum + hashmap independently; correct code with `long long`. Misses: stated O(n) but `std::map` made it O(n log n) (fixed when prompted); allocated an unnecessary full prefix array.

| Criterion | Score | Note |
|---|---|---|
| Problem understanding & clarification | 5/5 | Nailed negatives/zeros/k-sign upfront |
| Approach & thought process | 5/5 | Optimal structure, reasoned independently |
| Code quality & correctness | 4/5 | Correct + long long; minor std::map / spurious operator[] inserts |
| Complexity analysis | 4/5 | Claimed O(n) but std::map → O(n log n); fixed when prompted |
| Communication | 4/5 | Clear, traced when asked; could volunteer optimizations |

### Part 2 — Schema Design
Solid. Composite indexes from access patterns, denormalization with named historical reason, correct composite PK, knew partial indexes by name, sharp question on which column the partial index covers. Gaps: thin access-pattern enumeration (jumped to tables), forgot OrderItem link in first sketch, terse on tradeoffs until prompted.

| Criterion | Score | Note |
|---|---|---|
| Entity ID & requirements clarification | 3.5/5 | Good entities; access patterns thin until prompted |
| Table design (keys, types, relationships) | 4/5 | Solid Order table; missed OrderItem first pass, recovered |
| Indexing & query optimization | 4/5 | Composite + partial + GiST + denormalized pickup loc |
| Normalization & tradeoffs | 4/5 | Named denormalization & cost when pushed |
| Edge cases & data integrity | 3.5/5 | Price snapshot strong; skipped composite-PK break case |
| Communication | 3/5 | Terse — elaborate tradeoffs without being asked |

**Top takeaway:** Technically strong. Biggest lever: proactive elaboration — complexity *and why*, tradeoff *and its cost*, access patterns *before* tables.
