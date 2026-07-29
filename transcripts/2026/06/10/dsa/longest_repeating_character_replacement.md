# DSA Round Transcript
**Date:** 2026-06-10
**Start Time:** 8:34
**End Time:** 9:21
**Duration:** 47 minutes
**Problem:** Longest Repeating Character Replacement
**Topic:** Sliding Window
**Difficulty:** Medium

---

## Problem Statement
You are given a string `s` consisting of uppercase English letters, and an integer `k`. You can choose any character of the string and change it to any other uppercase English letter. You can perform this operation at most `k` times. Return the length of the longest substring containing the same letter you can get after performing the above operations.

**Constraints:**
- `1 <= s.length <= 10^5`
- `s` consists of only uppercase English letters
- `0 <= k <= s.length`

**Example 1:**
```
Input:  s = "ABAB", k = 2
Output: 4
Explanation: Replace the two 'A's with two 'B's, or vice versa -> "BBBB" or "AAAA".
```

**Example 2:**
```
Input:  s = "AABABBA", k = 1
Output: 4
Explanation: Replace the one 'A' in the middle "ABBA" -> "BBBB". Length 4.
```

---

## Conversation Log

**Interviewer:** Presented the problem with examples and constraints; asked for clarifying questions and high-level approach before coding.

**Aayush:** What are the constraints?

**Interviewer:** Provided constraints (`1 <= s.length <= 10^5`, uppercase A-Z only, `0 <= k <= s.length`; noted `k` can be 0).

**Aayush:** Looking at the constraints for the string size, the algorithm must be at max O(n).

**Interviewer:** Confirmed — good instinct to read the complexity ceiling off the constraints. Asked for the high-level approach.

**Aayush:** For any window of the string, as the window size increases there is a higher chance we require more operations to convert the window; as window size decreases, operations needed decreases. For a window [l:r], the min operations needed = (r-l+1) - maxFrequency of char in window. If this > k, shrink the window; else expand. Use l, r starting at 0, while loop r<n: increment freq of s[r], check if (r-l+1 - maxFreq) > k; if yes l++, else update ans = max(ans, window length) and move r ahead.

**Interviewer:** Confirmed core invariant is correct. Pressed on two points: (1) is it one `l++` or a `while` loop, and why; (2) how exactly is maxFreq computed and what does it cost.

**Aayush:** It would be a while loop; we iterate through the freq map which has at most 26 keys since letters are bounded.

**Interviewer:** Correct — O(26n) = O(n). Asked him to code it.

**Aayush:** (See solution below — naive sliding window with `getMaxFreq` scanning the 26-element array.) TC is O(n), SC is O(1) since freq array is just 26 in size.

**Interviewer:** Confirmed code is correct (traced "ABAA", k=0 -> 2). Pressed on precise complexity breakdown of getMaxFreq, and asked him to prove the inner while loop isn't O(n^2).

**Aayush:** O(26n) ~ O(n), 26 for getMax. Each index would be visited at most twice — once by r and once by l — so it is O(2n) ~ O(n).

**Interviewer:** Exactly right on both. Asked: can you optimize further — is the 26-scan necessary every iteration?

**Aayush:** Not sure how to avoid the scan. The movement of the pointers means the max keeps changing dynamically; we can't store one max and keep updating it when we encounter a new max because it would break when l moves and frequency decreases.

**Interviewer:** Gave a hint: the algorithm returns the longest valid window ever seen (only ever max'd). If maxFreq never decreases (historical max), can a stale-high max ever report a window LARGER than achievable?

**Aayush:** Not sure.

**Interviewer:** Revealed the optimization and the correctness reasoning (see Optimal Solution).

---

## Solution
**Aayush's Final Solution:**
```cpp
#include <bits/stdc++.h>
using namespace std;

int getMaxFreq(vector<int>& freqCnt)
{
    int mx = *(max_element(freqCnt.begin(), freqCnt.end()));
    return mx;
}
int main() {
    string s="ABAA";
    int k=0;
    vector<int> freq(26,0);
    int l = 0,r=0;
    int ans = 0;
    while(r<s.size())
    {
        freq[s[r]-'A']++;
        int maxFreq = getMaxFreq(freq);
        while(r-l+1 - maxFreq > k)
        {
            freq[s[l]-'A']--;
            l++;
            maxFreq = getMaxFreq(freq);
        }
        ans = max(ans, r-l+1);
        r++;
    }
    cout<<"Ans is "<<ans;
    return 0;
}
```

**Optimal Solution (if different):**
```cpp
int characterReplacement(string s, int k) {
    vector<int> freq(26, 0);
    int l = 0, maxFreq = 0, ans = 0;
    for (int r = 0; r < s.size(); r++) {
        freq[s[r] - 'A']++;
        maxFreq = max(maxFreq, freq[s[r] - 'A']);   // only ever grows (historical max)
        if ((r - l + 1) - maxFreq > k) {            // window invalid -> slide
            freq[s[l] - 'A']--;
            l++;
        }
        ans = max(ans, r - l + 1);
    }
    return ans;
}
```
**Why the stale max is safe:** `ans` only ever increases, and to record a NEW larger answer you need a window genuinely bigger than any seen before. When the window is invalid, l and r both advance by one, so the window size never shrinks — it slides. A stale-high maxFreq keeps the window the same size, so it can't manufacture a larger answer. The window only truly grows when a real character pushes maxFreq to a genuine new high. Hence the recorded answer is always achievable. This removes the 26-scan and the inner loop -> true O(n), O(1) work per step.

**Time Complexity:** Naive: O(26n) ~ O(n). Optimal: O(n).
**Space Complexity:** O(1) (26-element freq array).

---

## Feedback Given

**Problem Understanding & Clarification — 4/5**
Strong. Proactively asked for constraints before approaching, and read the complexity ceiling (10^5 -> ~O(n)) off the constraints — a senior instinct. Did not explicitly confirm s is strictly uppercase A-Z (assumed 26; happened to be right). Minor, worth verbalizing.

**Approach & Thought Process — 4.5/5**
Excellent. Derived the core invariant (window_size - maxFreq) <= k from first principles and articulated the monotonic intuition cleanly. Self-corrected from if to while for shrinking with correct justification. Strongest category today.

**Code Quality & Correctness — 4.5/5**
Clean, correct C++. Factored out getMaxFreq. Naive version bug-free and handled k=0 correctly. No correctness notes.

**Complexity Analysis — 5/5**
Real improvement. Named the true O(26n) factor for the freq scan AND gave the amortized two-pointer argument (each index visited twice, once by r once by l) to defend the inner loop against O(n^2). Precise and complete — historically a weak spot, so a genuine win.

**Communication — 3.5/5**
Good think-aloud. Gap: when asked to optimize, correctly identified the obstacle (stale max when l moves) but stopped at "not sure" after one hint rather than reasoning through why a stale max is harmless. The leap not made: the answer is monotonic (only ever max'd), so a stale-high max can only slide the window, never inflate the result. Build the muscle of attacking *why* an obstacle might be harmless rather than concluding it's fatal.

### Overall: ~4.3/5 — Strong round.
Naive O(n) solution complete and correct; complexity rigor was excellent. Growth edge: the historical-max optimization where monotonicity makes a stale value safe.

**Time Taken: 47 minutes**
