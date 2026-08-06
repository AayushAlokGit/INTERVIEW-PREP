# DSA Weaknesses
Last updated: 2026-08-06

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't proactively ask about input semantics (empty, duplicates) | 17 | 9 | 2026-08-06 |
| Asks for constraints but can't translate them to a budget | 11 | 6 | 2026-08-06 |
| Misses free structural facts stated in the problem | 6 | 2 | 2026-08-06 |
| Skips clarifying questions on value ranges / scale | 28 | 1 | 2026-07-23 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 25 | 8 | 2026-08-06 |
| Adopts an optimality principle without proving it | 5 | 3 | 2026-08-05 |
| Proposes sliding window without checking predicate monotonicity | 3 | 3 | 2026-08-05 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 78 | 10 | 2026-08-06 |
| Off-by-one / ordering bugs (update-before-shrink, mark-on-pop) | 17 | 4 | 2026-08-06 |
| Increment and decrement guards not mirror conditions | 3 | 2 | 2026-08-05 |
| Misses degenerate/identity edge cases (source == target) | 1 | 1 | 2026-08-06 |
| Ignores integer overflow implied by the given constraints | 1 | 1 | 2026-08-06 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't state complexity unless explicitly asked | 10 | 8 | 2026-08-05 |
| Doesn't check own complexity against the constraint budget | 2 | 2 | 2026-08-06 |
| States a confident bound that doesn't match his own code | 1 | 1 | 2026-08-06 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Long silence (7+ min) when stuck instead of thinking aloud | 14 | 10 | 2026-08-06 |
| Defends/asserts instead of tracing when asked to dry-run | 22 | 7 | 2026-08-06 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 14 | 10 | 2026-08-06 |
| No code written within the coding-phase budget | 13 | 10 | 2026-08-06 |

## Derivation Questions
<!-- Updated by /derive-optimal-algorithm. Ran = times he invoked the question unprompted
     when it was the one that mattered. Missed = times it was the unlocking question and he never reached it. -->
| # | Question | Ran | Missed | Last Missed |
|---|---|---|---|---|
| Q1 | Write the brute force as a function signature | 3 | 3 | 2026-08-04 |
| Q2 | Name the repeated work | 6 | 0 | — |
| Q3 | Fix the most constrained variable | 1 | 0 | — |
| Q4 | Is the predicate monotone? | 1 | 3 | 2026-08-05 |
| Q5 | Which scan direction/order makes it known? | 2 | 1 | 2026-07-28 |
| Q6 | Name the operation, match the structure | 1 | 2 | 2026-08-05 |
| Q7 | Candidate set too small, or move set too small? | 1 | 3 | 2026-07-29 |
| Q8 | What is the minimal state? | 1 | 2 | 2026-08-05 |
| Q9 | Which constraint have I not spent? | 0 | 4 | 2026-08-05 |
