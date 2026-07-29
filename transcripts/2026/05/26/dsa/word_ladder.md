# DSA Round Transcript
**Date:** 2026-05-26
**Start Time:** 10:29
**End Time:** 11:02
**Duration:** 33 minutes
**Problem:** Word Ladder
**Topic:** BFS / Graph / Hashing
**Difficulty:** Hard

---

## Problem Statement
Given two words `beginWord` and `endWord`, and a dictionary `wordList`, return the length of the shortest transformation sequence from `beginWord` to `endWord`, where:
1. Every adjacent pair of words differs by exactly one letter.
2. Every transformed word (except possibly beginWord) must exist in wordList.
3. Return 0 if no such sequence exists.

Length counts the words in the sequence including both endpoints.

**Constraints:**
- All words same length, lowercase English letters
- 1 <= beginWord.length <= 10
- 1 <= wordList.length <= 5000
- All entries distinct; beginWord may or may not be in wordList

---

## Conversation Log

**Interviewer:** Presented problem.

**Aayush:** Constraints? And does "differ by one letter" mean same length with one char different, or sizes differ by one?

**Interviewer:** Same length, exactly one position different. Provided constraints.

**Aayush:** Model as graph. Start = beginWord. Neighbors = words in dict differing by one char. BFS from beginWord tracking levels. If reach endWord, return level. Else 0.

**Interviewer:** Asked: how to find neighbors and cost? Overall TC/SC?

**Aayush:** Neighbor finding = |word| * |wordDict|. TC = O(|wordDict|² * |word|). SC = O(|wordDict| * |word|).

**Interviewer:** With N=5000, L=10, that's ~250M ops. Can you do better?

**Aayush:** Can you give a hint?

**Interviewer:** Hinted at wildcard patterns: for "hot" generate "*ot", "h*t", "ho*" and bucket words by pattern.

**Aayush:** Compute patterns for each word and union with dict.

**Interviewer:** Clarified: pre-compute pattern→words map once globally; during BFS look up patterns. Asked TC.

**Aayush:** O(NL²).

**Interviewer:** Correct. Code it up.

**Aayush:** [submitted clean BFS with pattern bucketing, beginWord inserted into dict, endWord-not-in-dict guard]

**Interviewer:** TC/SC?

**Aayush:** O(NL²) and O(N).

**Interviewer:** Pushed on SC — pattern map dominates.

**Aayush:** O(NL) for queue + set.

**Interviewer:** Patterns map has N*L entries with key length L. SC is O(NL²).

---

## Solution
**Aayush's Final Solution:**
```cpp
class Solution {
public:
    int ladderLength(string beginWord, string endWord, vector<string>& wordList) {
        unordered_set<string> dict(wordList.begin(), wordList.end());
        if (!dict.count(endWord)) return 0;

        int L = beginWord.size();
        unordered_map<string, vector<string>> patterns;
        dict.insert(beginWord);

        for (const string& word : dict) {
            for (int i = 0; i < L; i++) {
                string pattern = word;
                pattern[i] = '*';
                patterns[pattern].push_back(word);
            }
        }

        queue<pair<string, int>> q;
        unordered_set<string> visited;
        q.push({beginWord, 1});
        visited.insert(beginWord);

        while (!q.empty()) {
            auto [word, steps] = q.front(); q.pop();
            if (word == endWord) return steps;
            for (int i = 0; i < L; i++) {
                string pattern = word;
                pattern[i] = '*';
                for (const string& neighbor : patterns[pattern]) {
                    if (!visited.count(neighbor)) {
                        visited.insert(neighbor);
                        q.push({neighbor, steps + 1});
                    }
                }
            }
        }
        return 0;
    }
};
```

**Time Complexity:** O(N · L²)
**Space Complexity:** O(N · L²)

---

## Feedback Given

### Scoring (out of 5)

| Category | Score | Notes |
|---|---|---|
| Problem understanding & clarification | 5 / 5 | Strong clarifying questions including the same-length subtlety. |
| Approach & thought process | 3.5 / 5 | BFS natural; needed hint for wildcard bucketing trick. |
| Code quality & correctness | 5 / 5 | Clean, correct first try. Edge guards handled. |
| Complexity analysis | 2.5 / 5 | TC correct after hint. SC required two prompts to get the L² factor. |
| Communication | 4 / 5 | Clear narration. |

### Highlights
- Strong clarification habits this round.
- Clean correct BFS code, no debugging needed.

### Areas to work on
- Structure-exploiting tricks like wildcard bucketing — invert indexing when pair-wise comparison is too expensive.
- Space analysis on maps with derived/expanded keys — multiply by both number of derived keys and key size.
- Mention bidirectional BFS as optimization.

**Time Taken: 33 minutes**
