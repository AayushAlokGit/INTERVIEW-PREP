# DSA Weaknesses
Last updated: 2026-09-30

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Asks for constraints but can't translate them to a budget | 28 | 9 | 2026-09-29 |
| Doesn't proactively ask about input semantics (sorted, duplicates) | 32 | 8 | 2026-08-24 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Adopts an optimality principle without proving it | 19 | 10 | 2026-09-30 |
| Defaults to generic pattern over structure-exploiting one | 41 | 9 | 2026-09-29 |
| Takes the problem's operation phrasing as the algorithm axis | 5 | 3 | 2026-09-29 |
| Can't reduce a brute force without being told what to fix | 6 | 3 | 2026-09-29 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Tests only the given examples, never a self-made input | 16 | 10 | 2026-09-30 |
| Doesn't self-verify/dry-run before declaring done | 93 | 10 | 2026-09-30 |
| Declares a modulus but never reduces the accumulator/return | 1 | 1 | 2026-08-24 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't check own complexity against the constraint budget | 19 | 9 | 2026-09-29 |
| Declares "can't optimise" without checking auxiliary space | 7 | 5 | 2026-09-29 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Long silence (7+ min) when stuck instead of thinking aloud | 31 | 10 | 2026-09-30 |
| Defends/asserts instead of tracing when asked to dry-run | 33 | 9 | 2026-09-30 |
| Asks for a hint instead of attempting the question posed | 5 | 1 | 2026-09-29 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 31 | 10 | 2026-09-30 |
| No code written within the coding-phase budget | 30 | 10 | 2026-09-30 |

## Derivation Questions
<!-- Updated by /derive-optimal-algorithm. Ran = times he invoked the question unprompted
     when it was the one that mattered. Missed = times it was the unlocking question and he never reached it. -->
| # | Question | Ran | Missed | Last Missed |
|---|---|---|---|---|
| Q1 | Write the brute force as a function signature | 4 | 3 | 2026-08-04 |
| Q2 | Name the repeated work | 12 | 2 | 2026-08-24 |
| Q3 | Fix the most constrained variable | 2 | 2 | 2026-08-22 |
| Q4 | Is the predicate monotone? | 2 | 7 | 2026-08-24 |
| Q5 | Which scan direction/order makes it known? | 4 | 1 | 2026-07-28 |
| Q6 | Name the operation, match the structure | 6 | 6 | 2026-08-22 |
| Q7 | Candidate set too small, or move set too small? | 1 | 4 | 2026-08-06 |
| Q8 | What is the minimal state? | 1 | 4 | 2026-08-24 |
| Q9 | Which constraint have I not spent? | 1 | 16 | 2026-08-22 |
