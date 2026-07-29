# Derivation Drill Transcript
**Date:** 2026-07-29
**Start Time:** 18:23 · **End Time:** 18:41 · **Duration:** 18 min
**Problems:** Course Schedule III

## Scorecard
| Problem | Unlocking Q | Key move reached | Time | Edge | Unstated assumption | Internal state | Free swing |
|---|---|---|---|---|---|---|---|
| Course Schedule III | Q7 | N | 13/13 | M | M | M | M |

## Question Tally
| Q | Ran it? | Notes |
|---|---|---|
| Q1 | No | Never wrote a brute-force signature, so "which subset" was never named as the decision variable |
| Q2 | No | Never named the repeated work; search stalled at "sort and scan" |
| Q3 | n/a | Not the lever here |
| Q4 | n/a | Not a monotone-predicate problem |
| Q5 | **Yes** | Deadline ordering, with a correct exchange argument. Genuinely load-bearing |
| Q6 | No | Never said the per-step operation aloud; "report and remove the max taken" names the heap instantly |
| Q7 | **No — the unlocking question** | Never falsified his greedy, so never asked coverage-vs-expressiveness |
| Q8 | No | Not the lever, but the `time == sum(heap)` invariant is a state question |
| Q9 | No | `n <= 10^4` buys sort **plus** a log-factor structure; he spent the sort and stopped |

## Adversarial Tally
| Category | P1 | Session hit rate |
|---|---|---|
| Edge of the input space | M | 0/1 |
| Unstated assumption | M | 0/1 |
| Internal state | M | 0/1 |
| Free swing | M | 0/1 |

Distinct categories covered by his inputs: **0 of 4** (nothing submitted within the 180 s).

---

## Problem 1 — Course Schedule III
**Topic:** M6 / F1 (commit optimistically, then retract — regret heap) · **Originally solved:** 2026-05-11, Approach rated 3/5
**Presented as:** courses as `(duration, deadline)` pairs, one at a time from day 1, any order, may skip; maximise count finished on or before deadline. `n <= 10^4`, values `<= 10^4`. Example `[[100,200],[200,1300],[1000,1250],[2000,3200]] -> 3`.

**His derivation:**
> core idea is that we want to maximise the course possible so we will first pick courses which have an earlier deadline , this will allow us more room for courses with later deadline.
>
> sort by th deadline and scan from left to right. A course can be picked currentDedline + course duration <= courde deadline.
> if a course is picked currentDeadline advances by the courses duration
>
> in case of matching deadlines sort by duration

**Grade:**
- **Reached the key move?** **No.** The key move is the retraction: on overflow, compare against the longest course already taken and swap if that one is longer. Count preserved, elapsed time strictly decreases. His chain is take-if-it-fits with no undo.
- **Right:** the exchange argument that order within a chosen set is free, so sorting on deadline is safe. Step 2 of 5, and load-bearing — but it's the setup for the subset decision, not the decision.
- **Unlocking question:** Q7. Not run. His greedy fails on `[[5,5],[4,6],[2,6]]`: takes (5,5), 5+4=9>6 skip, 5+2=7>6 skip -> **1**; optimum (4,6) then (2,6) -> **2**. Move-set failure (missing `pop`), not coverage.
- **Skipped that would have helped:** Q1 (no signature), Q6 (never named the operation), Q9 (unspent log factor).
- The tie-break-by-duration addendum is a patch on a broken frame; with the swap move, ties are irrelevant.
- **Time:** 13 of 13, cut off.

**Correct chain:**
1. **Trigger:** any order, any subset, `n <= 10^4`. **Brute force:** `solve(i, timeUsed) -> max courses from courses[i..]`, 2^n subsets. Dead.
2. **Trigger:** for a fixed chosen set, if any order is feasible then deadline order is feasible (swap any inverted adjacent pair — neither breaks). **Move:** freeze the ordering by sorting on deadline; the only remaining decision is which subset.
3. **Trigger:** take-if-it-fits fails on `[[5,5],[4,6],[2,6]]`. **Diagnosis -> Q7:** move set too small, not candidate set — you only learn a long course was a bad purchase when a later short one arrives. **Move:** M6.
4. **Trigger:** swapping the longest taken course for a shorter overflowing one keeps the count and strictly reduces elapsed time — never worse for the future. **Move:** on overflow, if `maxTakenDuration > duration_i`, pop it and push `i`; else skip `i`.
5. **Trigger:** per-step operation is "report and remove the maximum among taken." **Move:** max-heap of taken durations. `O(n log n)` time, `O(n)` space.
6. **Invariant over the whole scan:** after the prefix ending at `k`, the heap is a maximum-size feasible set for that prefix and, among all maximum-size sets, has minimum total duration. Corollary: `time == sum(heap)` after every step.

**Adversarial inputs (attacking the correct chain):**
> *(nothing submitted within the 180 s)*
>
> Post-bell, after grading:
> 1. one course , output should be 1
> 2. all courses share the deadline -> in this case order depends on course duration.

**Adversarial grade:**
- Edge **Miss** · Unstated assumption **Miss** · Internal state **Miss** · Free swing **Miss**. Nothing submitted in time.
- Internal state: the mandatory variable list was never attempted — the category was never entered. Fifth consecutive Miss on this row.
- Nothing broke the chain because nothing was tried; no retroactive downgrade needed (derivation was already N).
- **Post-bell inputs, informal:** `n=1` is handled by the chain, and the predicted output is wrong — `[[5,3]]` must return **0**, not 1. That near-miss *is* the real edge case. "All courses share the deadline" is a class, not an input; with the swap move, duration order is irrelevant, so the chain handles it. Both score Miss regardless.
- **Input he should have found (Internal state):** carried variables are exactly two — `time` (elapsed days) and `heap` (taken durations) — and they are coupled: on a swap step **both must move in opposite directions within a single iteration** (`time += duration_i` and `time -= popped`). On `[[5,5],[4,6],[2,6]]`, a chain that pops the 5 and pushes the 4 but writes only one direction (subtracts the pop, forgets the push) leaves `time=0, heap={4}`; (2,6) is then accepted against a `time` understating reality by 4, and the algorithm reports **3** where the truth is **2**. Step 4 is responsible: it states the pop/push rule but never states that `time` is derived and must equal `sum(heap)` after every step. This is the same increment/decrement-mirror shape as the 07-28 `duplicates++/--` bug.
- **Time:** 180 s of 180, empty.

---

## Debrief

**Question tally.** Ran **Q5** unprompted, with a correct exchange argument — that's real. Never touched **Q1**, **Q6**, **Q7**, **Q9**. Q7 was the unlocking question and went untouched; fourth time it has been the gate. Q2, historically his reliable one, also didn't fire — he never named the repeated work, which is why the search stalled at "sort and scan."

The pattern: he produced a greedy and never tried to kill it. Every prior M6 failure (Min Refueling Stops, Course Schedule III, Furthest Building, Remove Duplicate Letters) has the same shape. Not a knowledge gap — he knows regret heaps. He stopped before the falsification step that would have demanded one.

**Adversarial tally.** 0/4, 0 distinct categories covered. **Internal state is now 0 hits in 5 attempts** — the only category that has never landed, welded to the largest open weakness. An empty round is worse than four wrong guesses: a wrong guess at least produces a concrete input to argue about.

**The one question to focus on next session — Q7.** The moment you have a greedy, before refining it, spend 60 seconds constructing an input where it loses. Can't build one in 60 s? Say why it's provably optimal. Found one? Ask "missing candidate or missing move?" — a greedy that picks a legal item and can't unpick it is *always* a missing move.

**Standing rule from the adversarial half:** write the variable list even with nothing else. Two lines — `time`, `heap` — takes ten seconds and is where the category gets won.
