# Derivation Drill Transcript
**Date:** 2026-07-29
**Start Time:** 18:55 · **End Time:** 19:09 · **Duration:** 14 min
**Problems:** Furthest Building You Can Reach

*(Second sitting of the day; `session_1823.md` covers Course Schedule III. This one ran under the
derivation-only version of the drill — the adversarial round was removed from the skill between the two.)*

## Scorecard
| Problem | Unlocking Q | Key move reached | Time |
|---|---|---|---|
| Furthest Building You Can Reach | Q7 | N | 13/13 |

## Question Tally
| Q | Ran it? | Notes |
|---|---|---|
| Q1 | **Yes** | `solve(i, bricks, ladders) -> furthest index`, `O(2^n)`, per-step choice named. Best opening he has written in this drill; he skipped this question 40 min earlier |
| Q2 | No | Never named the repeated work |
| Q3 | n/a | Not the lever |
| Q4 | n/a | Not a monotone-predicate problem |
| Q5 | n/a | Direction is forced by the problem |
| Q6 | No | "Report and remove the smallest ladder-assigned climb" names a min-heap instantly |
| Q7 | **No — the unlocking question** | Second consecutive miss, on the same family, 20 min after being told to run it |
| Q8 | No | Would have flagged that `bricks` cannot live in the state |
| Q9 | No | `bricks <= 10^9` forbids DP over brick count; `n <= 10^5` specifies `n log n` |

---

## Problem 1 — Furthest Building You Can Reach
**Topic:** M6 / F1 (commit optimistically, then retract — regret heap) · **Originally solved:** 2026-05-17
**Presented as:** `heights[]`, `bricks`, `ladders`; move `i -> i+1`; free if `h[i+1] <= h[i]`, else pay `h[i+1]-h[i]` bricks or one ladder; return furthest reachable index. `n <= 10^5`, `heights[i] <= 10^6`, `bricks <= 10^9`, `ladders <= n`. Example `[4,2,7,6,9,14,12]`, 5 bricks, 1 ladder -> 4.

**His derivation:**
> a simple brute force would be solve(i,bricks,ladders) -> the firthest index reachable from i using "bricks" and "ladders" The answer would be solve(0,...)
> At each index if h[i+1] is greater we have choice to use euther brick or ladder. TIme complexity is O(2^n)
>
> at each index i in order to move forward to index i+1 if h[i+1] > h[i] we use either bricks or ladders. Since ladders allow for more freedom in height difference prefer to use ladders only when bricks cant do the job.

**Grade:**
- **Reached the key move?** **No.** Key move: ladders must cover the `k` largest climbs encountered, but which are largest is only knowable in hindsight — so pay everything with bricks and retract via a min-heap of ladder-assigned climbs.
- **Real improvement:** Q1 ran properly, with a correct signature and complexity. That was the miss 40 minutes earlier.
- **Unlocking question:** Q7, not run. His rule fails on `heights = [1,11,12,13,14]`, `bricks = 10`, `ladders = 1`: 10 bricks go on the 1->11 climb, the ladder burns on a climb of 1, stalls at index **2**; optimum puts the ladder on the 10 and pays 1+1+1 with bricks -> index **4**. Move set too small, not candidate set.
- **Skipped:** Q6 (names the min-heap), Q9 (`bricks <= 10^9` kills any brick-indexed state — the tell that his own DP frame was unsalvageable).
- **The precise error:** "ladders allow more freedom, so save them" is directionally right, operationally backwards — ladder value scales with the climb it covers, so ladders belong on the *biggest* climbs. Recognising that the greedy needs hindsight *is* the retraction trigger. He stopped one step short.
- **Time:** 13 of 13, cut off.

**Correct chain:**
1. **Trigger:** two resource types, binary choice per climb. **Brute force:** `solve(i, bricks, ladders) -> furthest index`, `O(2^n)`.
2. **Trigger -> Q9:** `bricks <= 10^9`, so no state can be indexed by brick count. The DP frame is dead, not merely slow.
3. **Trigger:** only positive climbs cost anything, and a ladder is worth exactly the climb it covers. **Move (M9):** rewrite the objective — not "brick or ladder at step `i`" but "which `ladders`-sized subset of the climbs so far goes to ladders", the rest summing to `<= bricks`. For a fixed prefix, that subset is the `ladders` largest climbs.
4. **Trigger:** the `ladders` largest climbs of a prefix aren't knowable while scanning it — a later climb can displace an earlier choice. **Diagnosis -> Q7:** move set too small; I need to un-assign a ladder. **Move:** M6.
5. **Trigger:** per-step operation is "report and remove the smallest ladder-assigned climb." **Move (Q6):** min-heap of size `ladders`. Per climb `d > 0`: push `d`; if `size > ladders`, pop the min and `bricks -= popped`; if `bricks < 0`, return `i`. `O(n log k)`.
6. **Invariant over the whole scan:** on reaching index `i`, the heap holds exactly the `min(ladders, #climbs)` largest climbs in `[0..i]` and `bricks` reflects the sum of the rest — the minimum possible brick spend for reaching `i`.

---

## Debrief

**Question tally.** Q1 ran, and well — that is a genuine gain over the 18:23 sitting where it went untouched. Q7 missed again. Q6 and Q9 untouched for the second consecutive problem.

**Cross-problem connection — the headline.** This is the same skeleton as Course Schedule III, drilled 40 minutes earlier: *forced arrival order + a cumulative budget + a commitment you can undo* -> take everything, evict the worst commitment from a heap when the budget breaks. Different nouns, one algorithm. He missed Q7 on both, the second time immediately after being told Q7 was the thing to run. Retrieval failure, not a knowledge gap — the fifth instance of the M6 pattern in the record (Min Refueling Stops 05/06, Course Schedule III 05/11, Furthest Building 05/17, Remove Duplicate Letters 07/28, and today twice).

**The one question to focus on next session — Q7, mechanised.** The abstract instruction didn't transfer across 40 minutes, so it needs a syntactic trigger rather than a conceptual one: *the moment you write a greedy rule containing the word "prefer" or "only when", stop and spend 60 seconds building an input where it loses. If the loss traces to a commitment you can't take back, the answer is a heap.*

**Selection note for next time:** stop serving F1. He can derive the regret heap when he reaches the falsification step; more F1 problems only re-measure the same miss. Serve the falsification habit inside a different family — M4 (38%) or M12 (50%) — so Q7 has to fire without the family cueing it.
