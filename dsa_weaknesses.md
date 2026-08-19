# DSA Weaknesses
Last updated: 2026-08-18

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't proactively ask about input semantics (empty, duplicates) | 22 | 10 | 2026-08-18 |
| Asks for constraints but can't translate them to a budget | 17 | 10 | 2026-08-18 |
| Misses free structural facts stated in the problem | 11 | 6 | 2026-08-18 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 31 | 10 | 2026-08-18 |
| Adopts an optimality principle without proving it | 9 | 5 | 2026-08-18 |
| Proposes sliding window without checking predicate monotonicity | 3 | 1 | 2026-08-05 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 84 | 10 | 2026-08-18 |
| Off-by-one / ordering bugs (update-before-shrink, mark-on-pop) | 21 | 7 | 2026-08-18 |
| Tests only the given examples, never a self-made input | 5 | 5 | 2026-08-18 |
| Binary search: lo=mid paired with a down-biased mid | 1 | 4 | 2026-08-17 |
| Inverts a derived inequality when translating it into code | 1 | 1 | 2026-08-18 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't check own complexity against the constraint budget | 8 | 8 | 2026-08-18 |
| Doesn't state complexity unless explicitly asked | 13 | 8 | 2026-08-17 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Long silence (7+ min) when stuck instead of thinking aloud | 19 | 10 | 2026-08-18 |
| Defends/asserts instead of tracing when asked to dry-run | 26 | 9 | 2026-08-17 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 19 | 10 | 2026-08-18 |
| No code written within the coding-phase budget | 18 | 10 | 2026-08-18 |

## Derivation Questions
<!-- Updated by /derive-optimal-algorithm. Ran = times he invoked the question unprompted
     when it was the one that mattered. Missed = times it was the unlocking question and he never reached it. -->
| # | Question | Ran | Missed | Last Missed |
|---|---|---|---|---|
| Q1 | Write the brute force as a function signature | 3 | 3 | 2026-08-04 |
| Q2 | Name the repeated work | 7 | 1 | 2026-08-14 |
| Q3 | Fix the most constrained variable | 1 | 0 | — |
| Q4 | Is the predicate monotone? | 2 | 3 | 2026-08-05 |
| Q5 | Which scan direction/order makes it known? | 2 | 1 | 2026-07-28 |
| Q6 | Name the operation, match the structure | 2 | 4 | 2026-08-17 |
| Q7 | Candidate set too small, or move set too small? | 1 | 4 | 2026-08-06 |
| Q8 | What is the minimal state? | 1 | 3 | 2026-08-17 |
| Q9 | Which constraint have I not spent? | 0 | 7 | 2026-08-17 |
