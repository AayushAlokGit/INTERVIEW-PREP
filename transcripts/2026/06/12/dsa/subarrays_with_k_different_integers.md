# DSA Round Transcript
**Date:** 2026-06-12
**Start Time:** 10:53
**End Time:** 11:18
**Duration:** 25 minutes
**Problem:** Subarrays with K Different Integers
**Topic:** Sliding Window (atMost decomposition)
**Difficulty:** Hard

---

## Problem Statement
Given an integer array `nums` and an integer `k`, return the number of **good subarrays** — contiguous subarrays where the number of **distinct** integers is **exactly** `k`.

**Example 1:** `nums = [1,2,1,2,3], k = 2` → `7`
(`[1,2], [2,1], [1,2], [2,3], [1,2,1], [2,1,2], [1,2,1,2]`)

**Example 2:** `nums = [1,2,1,3,4], k = 3` → `3`
(`[1,2,1,3], [2,1,3], [1,3,4]`)

**Constraints:**
- `1 <= nums.length <= 2 * 10^4`
- `1 <= nums[i], k <= nums.length`

---

## Conversation Log

**Interviewer:** Presented problem, asked for clarifying questions / approach.

**Aayush:** What are the constraints?

**Interviewer:** `1 <= nums.length <= 2*10^4`, `1 <= nums[i], k <= nums.length`.

**Aayush:** Sliding window with a hashmap counting frequency of distinct elements and size. When window has cnt <= k increment r; if cnt == k increment ans; while cnt > k increment l.

**Interviewer:** For a fixed right pointer r, how many subarrays ending at r have exactly k distinct? Is it always 1? Trace at r=3 (window `[1,2,1,2]`).

**Aayush:** (Gave code) `while(r<n){ freq[nums[r]]++; while(freq.size()>k) freq[nums[l]]--, l++; if(freq.size()==k) ans++; r++; }`

**Interviewer:** You skipped the question. After r=3, window is `[1,2,1,2]`, size==k, code adds 1. Enumerate by hand: how many subarrays ending at index 3 have exactly 2 distinct?

**Aayush:** There is only one subarray with 2 distinct ending at index 3, i.e. [2,3].

**Interviewer:** [2,3] ends at index 4, not 3. Index 3 is value 2. Subarrays ending at index 3: [2]=1 distinct, [1,2]=2 ✓, [2,1,2]=2 ✓, [1,2,1,2]=2 ✓ → that's 3, but your code added only 1. Counting "exactly k" with one window undercounts. Can you express "exactly k" via an easier quantity?

**Aayush:** No, my code would add 1 for [1,2], [1,2,1], [1,2,1,2] as r reaches indices 1,2,3 — so 1 added at each index.

**Interviewer:** Run your exact code end-to-end on `[1,2,1,2,3], k=2`. Report freq, l, ans after each r. What's the final ans vs expected 7?

**Aayush:** Answer comes out as 4 instead of 7, because at a given r we are not getting all the possible l's.

**Interviewer:** Right — your window only finds the longest valid l for each r. Now: count subarrays with AT MOST k distinct. For the longest valid window [l,r], how many subarrays ending at r have at most k distinct?

**Aayush:** r - l + 1.

**Interviewer:** Exactly — sum that over all r for atMost(k) in O(n). Now bridge to "exactly k": how do you get it from "at most" quantities?

**Aayush:** atMost(k) - atMost(k-1).

**Interviewer:** That's the trick. Write the full solution.

**Aayush:** (Wrote two-pass atMost helper with subtraction — correct.)

**Interviewer:** Correct — traced atMostK=12, atMostK1=5, 12-5=7. Now tighten complexity: you used std::map — what's it implemented as and what does each op cost? And how many keys does freq actually hold?

**Aayush:** I would use unordered_map to make it O(1) amortized / O(n) worst case, but with red-black-tree map I always get O(log n). freq holds the number of distinct keys, which is at most k.

**Interviewer:** Precise — std::map → O(log k) per op → solution is O(n log k); unordered_map → O(n). Space is O(k), not O(n). Excellent.

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int atMost(const vector<int>& nums, int k) {
  unordered_map<int,int> freq;   // use unordered_map for O(n)
  int l = 0, count = 0;
  for (int r = 0; r < (int)nums.size(); r++) {
    freq[nums[r]]++;
    while ((int)freq.size() > k) {
      if (--freq[nums[l]] == 0) freq.erase(nums[l]);
      l++;
    }
    count += r - l + 1;
  }
  return count;
}

int subarraysWithKDistinct(vector<int>& nums, int k) {
  return atMost(nums, k) - atMost(nums, k - 1);
}

int main() {
  vector<int> nums{1,2,1,2,3};
  cout << "ans is " << subarraysWithKDistinct(nums, 2);  // 7
  return 0;
}
```
*(Aayush's submitted version inlined two near-identical passes with `std::map`; structure and result identical — refactored above for clarity.)*

**Optimal Solution:** Same approach (atMost decomposition with sliding window) — this is optimal.

**Time Complexity:** O(n) with `unordered_map` (O(n log k) as submitted with `std::map`)
**Space Complexity:** O(k)

---

## Feedback Given

A tale of two halves: a rocky start, but a sharp finish.

**What went well:**
- Constraints asked first, again — two rounds in a row.
- Complexity analysis excellent: identified std::map as a red-black tree (O(log k) per op → O(n log k)), proposed unordered_map for true O(n), and corrected space to O(k). Most precise he's been on a chronic weak area.
- Clean, correct final code once on track, with proper erase-on-zero handling.

**What to sharpen:**
- Defended a wrong approach instead of tracing it. When asked to enumerate subarrays ending at index 3, answered `[2,3]` (misread the index), then insisted his code "adds up to 3 correctly" without executing. Only forced execution revealed it produces 4, not 7. Recurring pattern: when challenged, argues the code is right rather than running it. Trace first, claim second.
- Didn't reach the structure-exploiting insight independently. The `exactly(k) = atMost(k) - atMost(k-1)` decomposition needed heavy steering; his first instinct was to force a single window to count "exactly k" directly — the classic trap.

**Scoring (1–5):**
| Dimension | Score | Notes |
|---|---|---|
| Problem understanding & clarification | 4 | Constraints asked upfront |
| Approach & thought process | 3 | Initial counting logic wrong; needed heavy guidance to atMost decomposition |
| Code quality & correctness | 4 | Final solution correct and clean |
| Complexity analysis | 5 | Precise on log factor and O(k) space |
| Communication | 3 | Defended instead of tracing; misread indices under pressure |

**Overall: ~3.7/5.** Complexity rigor genuinely leveled up. The thing holding this round back: when someone questions your code, the first move should be to execute it, not defend it.

**Time Taken: 25 minutes**
