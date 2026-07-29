# DSA Round Transcript
**Date:** 2026-06-11
**Start Time:** 11:08
**End Time:** 11:34
**Duration:** 26 minutes
**Problem:** Top K Frequent Elements
**Topic:** Hash Map + Heap / Bucket Sort
**Difficulty:** Medium

---

## Problem Statement
Given an integer array `nums` and an integer `k`, return the `k` most frequent elements. Answer may be in any order.

**Example 1:** `nums = [1,1,1,2,2,3], k = 2` → `[1,2]`
**Example 2:** `nums = [1], k = 1` → `[1]`

**Constraints:** `1 <= nums.length <= 10^5`; `-10^4 <= nums[i] <= 10^4`; `k` in `[1, #unique]`; answer guaranteed unique.

---

## Conversation Log

**Interviewer:** Present problem; ask for clarifications.

**Aayush:** What are the constraints on k and nums?

**Interviewer:** (gives constraints above). Approach?

**Aayush:** Naive: hashmap of counts, convert to vector of (count, element) pairs, sort by count, return last k. TC O(n log n) for sort, O(n) for hashmap.

**Interviewer:** Correct. Can you do better than O(n log n)?

**Aayush:** Maintain a min-heap of size k and a frequency hashmap. Iterate through the array; while heap size > k, pop; then push {freq[nums[i]], nums[i]}. At the end the heap holds the k most frequent. TC reduces to O(n log k).

**Interviewer:** Pin down what you iterate over — you said "the array." Trace nums=[1,1,1,2,2,3], k=2 pushing per array element. What's in the heap?

**Aayush:** The same number would be pushed multiple times, but each time we'd pop the duplicates with the size < k check.

**Interviewer:** (traces) Final heap = two copies of element 1; element 2 evicted → wrong. The pop doesn't dedupe. Actual fix?

**Aayush:** We'd incrementally push frequency — first 1 push {1,1}, second push {2,1}, etc.

**Interviewer:** (traces [1,1,1,2,2], k=2) Same element still occupies multiple slots → could output [1,1]. Root problem: one element in more than one heap slot. What would you change so each distinct element is pushed exactly once?

**Aayush:** Iterate over hashmap entries and push those to the heap.

**Interviewer:** Correct. Code it.

**Aayush:** (see Solution below)

**Interviewer:** Confirm it prints 2 and 1 — correct. Now full complexity, including building freq. Be precise.

**Aayush:** O(n) for freq and O(m log k) for the heap, where m = number of unique elements.

**Interviewer:** You used `map<int,int>`, not `unordered_map`. Is insertion into std::map O(1)? What does that do to "O(n) for freq"?

**Aayush:** I'd use unordered_map and make it O(1); ignore that detail.

**Interviewer:** (notes: with map it's O(n log m); don't hand-wave on a whiteboard). Total with unordered_map: O(n + m log k), space O(m). Can you beat O(m log k) and reach linear O(n)?

**Aayush:** Could you give a hint, I'm not getting it.

**Interviewer:** Think about the range of possible frequency values — smallest/largest frequency. Could you index by frequency directly?

**Aayush:** Yes — another array of size n+1, arr[i] = elements that appear i times. Frequency is bounded by n. One pass from index n down to 1 gives the k elements.

**Interviewer:** Can two different elements share a frequency? What should arr[i] hold?

**Aayush:** arr[i] is itself a vector, so arr is a vector of vectors of size n+1.

**Interviewer:** Correct — O(n) time, O(n) space, optimal.

---

## Solution
**Aayush's Final Solution (heap, O(n + m log k)):**
```cpp
unordered_map<int,int> freq;            // (he wrote map; corrected to unordered_map)
for(int i:nums) freq[i]++;
priority_queue<pair<int,int>, vector<pair<int,int>>, greater<pair<int,int>>> minH;
for(auto [num,cnt]:freq){
  minH.push({cnt,num});
  while(minH.size() > k) minH.pop();
}
// heap now holds the k most frequent
while(!minH.empty()){ auto [cnt,num]=minH.top(); minH.pop(); /* num is an answer */ }
```

**Optimal Solution (bucket sort, O(n)):**
```cpp
unordered_map<int,int> freq;
for(int x:nums) freq[x]++;
int n = nums.size();
vector<vector<int>> bucket(n+1);
for(auto [num,cnt]:freq) bucket[cnt].push_back(num);
vector<int> res;
for(int f=n; f>=1 && (int)res.size()<k; f--)
  for(int num:bucket[f]){ res.push_back(num); if((int)res.size()==k) break; }
return res;
```

**Time Complexity:** Heap: O(n + m log k). Bucket: O(n).
**Space Complexity:** O(m) heap version, O(n) bucket version.

---

## Feedback Given

**Time Taken: 26 minutes**

### What went well
- Naive hashmap + sort stated fast with correct O(n log n).
- Reached for min-heap of size k (O(n log k)) without prompting.
- Converged on optimal bucket-sort O(n) once hinted; correctly handled multiple-elements-per-frequency (vector of vectors).

### What needs work
- **Two buggy heap variants, both defended instead of traced.** v1 pushed per array element; when flagged, asserted "the pop removes duplicates" (it doesn't). v2 ("incremental frequencies") had the same one-element-multiple-slots flaw. Both times reasoned about intent, not actual behavior; interviewer had to dry-run [1,1,1,2,2] for him. Tracing his own first version would have caught it in ~60s.
- **Mis-attributed log factor again.** Wrote `std::map` (O(log m) insert) but claimed "O(n) for freq" = actually O(n log m). Waved it off as "ignore that detail." Claim didn't match the code written. Read your own data structure before stating complexity.

### Scoring
| Criterion | Score (/5) | Notes |
|---|---|---|
| Problem understanding & clarification | 4 | Good constraint question up front |
| Approach & thought process | 4 | Strong instincts; reached heap and bucket sort |
| Code quality & correctness | 3 | Final code correct, but two buggy heap designs preceded it |
| Complexity analysis | 3 | map vs unordered_map log factor mis-stated and waved off |
| Communication | 3 | Asserted correctness twice instead of tracing; interviewer did the dry-runs |

**Theme (recurring):** Algorithmic ideas are good, but he defends his design instead of stress-testing it. When an interviewer questions the approach, trace it on the example rather than restating why it should work — that would have caught both heap bugs himself.
