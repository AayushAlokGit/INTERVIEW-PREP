# DSA Weaknesses
Last updated: 2026-08-04

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't proactively ask about input semantics (empty, duplicates) | 14 | 10 | 2026-08-04 |
| Skips clarifying questions on value ranges / scale | 28 | 6 | 2026-07-23 |
| Asks for constraints but can't translate them to a budget | 8 | 5 | 2026-08-04 |
| Misses free structural facts stated in the problem | 4 | 3 | 2026-08-04 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 23 | 9 | 2026-08-04 |
| Adopts a greedy without an exchange/monotonicity argument | 3 | 3 | 2026-08-04 |
| Requests hints instead of one more independent attempt | 2 | 2 | 2026-08-04 |
| Proposes sliding window without checking predicate monotonicity | 2 | 2 | 2026-07-28 |
| Fails to recognise a technique he has already used | 3 | 1 | 2026-08-02 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 73 | 10 | 2026-08-04 |
| Off-by-one / ordering bugs (update-before-shrink, boundaries) | 16 | 7 | 2026-08-02 |
| Submits a solution that exceeds the stated constraint budget | 1 | 1 | 2026-08-04 |
| Writes recurrence missing the boundary/coupling term | 1 | 1 | 2026-07-28 |
| Increment and decrement guards not mirror conditions | 2 | 1 | 2026-07-28 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't state complexity unless explicitly asked | 7 | 7 | 2026-08-04 |
| Asserts best-case bound, omits true factor (descent/search) | 11 | 4 | 2026-07-23 |
| Proves invariant only within a loop, not across iterations | 4 | 3 | 2026-08-02 |
| Bounds the structure he built by the input size, not its own | 2 | 2 | 2026-08-04 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Long silence (7+ min) when stuck instead of thinking aloud | 11 | 10 | 2026-08-04 |
| Defends/asserts instead of tracing when asked to dry-run | 20 | 8 | 2026-08-02 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 11 | 10 | 2026-08-04 |
| No code written within the coding-phase budget | 10 | 9 | 2026-08-04 |

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
| Q9 | Which constraint have I not spent? | 0 | 3 | 2026-08-04 |
