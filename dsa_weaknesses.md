# DSA Weaknesses
Last updated: 2026-08-04

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't proactively ask about input semantics (empty, duplicates) | 13 | 10 | 2026-08-04 |
| Asks for constraints but can't translate them to a budget | 8 | 6 | 2026-08-04 |
| Skips clarifying questions on value ranges / scale | 28 | 7 | 2026-07-23 |
| Misses free structural facts stated in the problem | 3 | 2 | 2026-08-04 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 23 | 10 | 2026-08-04 |
| Adopts a greedy without an exchange/monotonicity argument | 2 | 2 | 2026-08-04 |
| Fails to recognise a technique he has already used | 3 | 2 | 2026-08-02 |
| Proposes sliding window without checking predicate monotonicity | 2 | 2 | 2026-07-28 |
| Requests hints instead of one more independent attempt | 1 | 1 | 2026-08-04 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 72 | 10 | 2026-08-02 |
| Off-by-one / ordering bugs (update-before-shrink, boundaries) | 16 | 8 | 2026-08-02 |
| Writes recurrence missing the boundary/coupling term | 1 | 1 | 2026-07-28 |
| Increment and decrement guards not mirror conditions | 2 | 1 | 2026-07-28 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't state complexity unless explicitly asked | 6 | 6 | 2026-08-04 |
| Asserts best-case bound, omits true factor (descent/search) | 11 | 5 | 2026-07-23 |
| Proves invariant only within a loop, not across iterations | 4 | 4 | 2026-08-02 |
| Miscounts space: calls O(1) auxiliary space O(n) | 1 | 1 | 2026-08-02 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Long silence (7+ min) when stuck instead of thinking aloud | 10 | 9 | 2026-08-04 |
| Defends/asserts instead of tracing when asked to dry-run | 20 | 9 | 2026-08-04 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 10 | 9 | 2026-08-04 |
| No code written within the coding-phase budget | 9 | 8 | 2026-08-04 |
| Abandons round instead of submitting a partial | 1 | 1 | 2026-08-04 |

## Derivation Questions
<!-- Updated by /derive-optimal-algorithm. Ran = times he invoked the question unprompted
     when it was the one that mattered. Missed = times it was the unlocking question and he never reached it. -->
| # | Question | Ran | Missed | Last Missed |
|---|---|---|---|---|
| Q1 | Write the brute force as a function signature | 2 | 3 | 2026-08-04 |
| Q2 | Name the repeated work | 5 | 0 | — |
| Q3 | Fix the most constrained variable | 1 | 0 | — |
| Q4 | Is the predicate monotone? | 1 | 2 | 2026-08-02 |
| Q5 | Which scan direction/order makes it known? | 2 | 1 | 2026-07-28 |
| Q6 | Name the operation, match the structure | 0 | 1 | 2026-07-28 |
| Q7 | Candidate set too small, or move set too small? | 1 | 3 | 2026-07-29 |
| Q8 | What is the minimal state? | 0 | 1 | 2026-08-04 |
| Q9 | Which constraint have I not spent? | 0 | 2 | 2026-08-04 |
