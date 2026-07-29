# DSA Round Transcript
**Date:** 2026-05-10
**Start Time:** 8:28
**End Time:** 9:05
**Duration:** 37 minutes
**Problem:** Alien Dictionary
**Topic:** Graphs, Topological Sort
**Difficulty:** Hard

---

## Problem Statement

You are given a list of `words` from an alien language's dictionary, sorted in lexicographic order according to that language's alphabet. The alien alphabet uses the same letters as English but in a different order.

Return a string of the unique letters in the alien language sorted by the rules of that language. If there are multiple valid answers, return any. If no valid ordering exists, return `""`.

**Constraints:**
- 1 <= words.length <= 100
- 1 <= words[i].length <= 100
- words[i] consists of only lowercase English letters

**Example 1:**
```
words = ["wrt", "wrf", "er", "ett", "rftt"]
Output: "wertf"
```

**Example 2:**
```
words = ["z", "x"]
Output: "zx"
```

**Example 3:**
```
words = ["z", "x", "z"]
Output: ""
```

---

## Conversation Log

**Interviewer:** Please note your start time, then here is your problem: [Alien Dictionary problem presented]

**Aayush:** 8:28

**Aayush:** What are the constraints?

**Interviewer:** 1 <= words.length <= 100, 1 <= words[i].length <= 100, words[i] consists of only lowercase English letters.

**Aayush:** I need to use the words array to construct the order of the alien alphabets and then if they form a DAG, the required order would be the topological sort of the DAG. However if no DAG is formed then return empty string. The real challenge here would be to construct the DAG from the words array. We can compare adjacent words to determine which letter comes earlier. We can have an edge from earlier letter to later coming letter. We can have 2 indices i and j and we stop when word1[i] != word2[j], now we can tell the order of the 2 chars of the word.

**Interviewer:** Correct. One edge case to flag before you code — what happens when comparing ["abc", "ab"]? Does that produce a valid edge, and what should you return?

**Aayush:** abc should not be before ab since abc has 3 chars while ab has only 2. [Implied: return ""]

**Interviewer:** Exactly right — if a longer word is a prefix of a shorter word and appears before it, that's an invalid ordering and you should return "". Now go ahead and code it up.

**Aayush:** [Submitted Version 1 — see Solution section]

**Interviewer:** Good structure. Before I give feedback: (1) What's the time and space complexity? (2) Try test case ["abc", "bcd"] mentally — what does your code output? Is that correct?

**Aayush:** [Submitted Version 2 with nodes fix — see Solution section]

**Interviewer:** Good catch — you fixed the missing nodes bug. Time and space complexity?

**Aayush:** Time complexity is O(|words| * max|words[i]|) and SC is bounded by number of lowercase English alphabets i.e. 26.

**Interviewer:** Correct on both. Now trace through ["ab", "ac", "db", "dc"] — what does your code output?

**Interviewer:** [Explained duplicate edge bug — "db" vs "dc" adds b→c again, inflating indeg[c] to 2, causing c to never be enqueued, returning "" instead of a valid ordering like "abdc".]

**Aayush:** 9:05

---

## Solution

**Aayush's Version 1 (buggy — missing nodes):**
```cpp
#include <bits/stdc++.h>
using namespace std;

bool constructAndContinue(string s1,string s2, map<char,vector<char>> &graph, map<char,int> &indeg, set<char> &nodes)
{
    if(s2.size() < s1.size() && s1.substr(0,s2.size()) == s2)
        return false;
    int i=0,j=0;
    while(i<s1.size() && j<s2.size())
    {
        if(s1[i]==s2[j]) { i++; j++; }
        else
        {
            graph[s1[i]].push_back(s2[j]);
            indeg[s2[j]]++;
            nodes.insert(s1[i]);
            nodes.insert(s2[j]);
            break;
        }
    }
    return true;
}
int main() {
    vector<string> words {"z", "x", "z"};
    int n = words.size();
    map<char,vector<char>> graph;
    map<char,int> indeg;
    set<char> nodes;
    string ans = "";
    queue<char> q;
    for(int i=1;i<n;i++) {
        bool shouldContinue = constructAndContinue(words[i-1], words[i], graph, indeg, nodes);
        if(!shouldContinue) { cout<<"ans is "<<ans<<endl; return 0; }
    }
    for(char ch:nodes)
        if(indeg.find(ch) == indeg.end()) q.push(ch);
    while(!q.empty()) {
        char ch = q.front(); q.pop();
        for(auto nbr:graph[ch]) { indeg[nbr]--; if(indeg[nbr] == 0) q.push(nbr); }
        ans += ch;
    }
    if(ans.size() != nodes.size()) { cout<<"ans is "<<endl; return 0; }
    cout<<"ans is "<<ans<<endl;
    return 0;
}
```

**Aayush's Final Version 2 (nodes fixed, duplicate edge bug remains):**
```cpp
#include <bits/stdc++.h>
using namespace std;

bool constructAndContinue(string s1,string s2, map<char,vector<char>> &graph, map<char,int> &indeg, set<char> &nodes)
{
    if(s2.size() < s1.size() && s1.substr(0,s2.size()) == s2)
        return false;
    int i=0,j=0;
    while(i<s1.size() && j<s2.size())
    {
        if(s1[i]==s2[j]) { i++; j++; }
        else
        {
            graph[s1[i]].push_back(s2[j]);
            indeg[s2[j]]++;
            break;
        }
    }
    i=0; j=0;
    while(i<s1.size() && j<s2.size()) { nodes.insert(s1[i++]); nodes.insert(s2[j++]); }
    while(i<s1.size()) nodes.insert(s1[i++]);
    while(j<s2.size()) nodes.insert(s2[j++]);
    return true;
}
int main() {
    vector<string> words {"z", "x", "z"};
    int n = words.size();
    map<char,vector<char>> graph;
    map<char,int> indeg;
    set<char> nodes;
    string ans = "";
    queue<char> q;
    for(int i=1;i<n;i++) {
        bool shouldContinue = constructAndContinue(words[i-1], words[i], graph, indeg, nodes);
        if(!shouldContinue) { cout<<"ans is "<<ans<<endl; return 0; }
    }
    for(char ch:nodes)
        if(indeg.find(ch) == indeg.end()) q.push(ch);
    while(!q.empty()) {
        char ch = q.front(); q.pop();
        for(auto nbr:graph[ch]) { indeg[nbr]--; if(indeg[nbr] == 0) q.push(nbr); }
        ans += ch;
    }
    if(ans.size() != nodes.size()) { cout<<"ans is "<<endl; return 0; }
    cout<<"ans is "<<ans<<endl;
    return 0;
}
```

**Optimal Fix (use set in adjacency list to prevent duplicate edges):**
```cpp
map<char, set<char>> graph; // set prevents duplicate edges automatically
// When adding edge:
if (graph[s1[i]].find(s2[j]) == graph[s1[i]].end()) {
    graph[s1[i]].insert(s2[j]);
    indeg[s2[j]]++;
}
```

**Time Complexity:** O(n · L) where n = number of words, L = max word length
**Space Complexity:** O(26) — bounded by alphabet size

---

## Feedback Given

### Problem Understanding & Clarification — 3/5
Asked for constraints (good). Did not ask about: identical adjacent words, single-word input, or whether all characters are guaranteed to appear in an ordering. The "skips clarifying questions" pattern continues.

### Approach & Thought Process — 5/5
Clean and correct. Immediately identified topological sort, correct graph construction strategy (compare adjacent words, first differing char gives edge), and the prefix invalidity case. Solid.

### Code Quality & Correctness — 3/5
- Version 1 bug: Missing nodes (characters not involved in any edge). Self-corrected after being prompted with a test case.
- Version 2 bug: Duplicate edges inflating indegree. Did not catch unprompted — needed a concrete test case to surface it.

### Complexity Analysis — 5/5
Correct. O(n·L) time, O(26) space. No issues.

### Communication — 3/5
Approach explanation was clear. Did not volunteer edge cases proactively — both bugs were found through interviewer-prompted tracing.

### Overall: 19/25

**Strengths:** Strong graph/algorithm intuition. Correct approach with no major conceptual gaps. Good complexity reasoning.

**To improve:** Before saying done, trace at least 2-3 edge cases: duplicate edges, single node, all identical words. Interviewers notice when candidates catch their own bugs.
