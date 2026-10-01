# DSA Weaknesses
Last updated: 2026-10-01

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't proactively ask about input semantics (sorted, duplicates) | 33 | 8 | 2026-10-01 |
| Asks for constraints but can't translate them to a budget | 28 | 7 | 2026-09-29 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Adopts an optimality principle without proving it | 21 | 10 | 2026-10-01 |
| Defaults to generic pattern over structure-exploiting one | 42 | 9 | 2026-10-01 |
| Takes the problem's operation phrasing as the algorithm axis | 6 | 3 | 2026-10-01 |
| Can't reduce a brute force without being told what to fix | 7 | 3 | 2026-10-01 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Tests only the given examples, never a self-made input | 18 | 10 | 2026-10-01 |
| Doesn't self-verify/dry-run before declaring done | 95 | 10 | 2026-10-01 |
| Arithmetic slips in hand traces go unnoticed | 1 | 1 | 2026-10-01 |
| Declares a modulus but never reduces the accumulator/return | 1 | 1 | 2026-08-24 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't check own complexity against the constraint budget | 21 | 10 | 2026-10-01 |
| Declares "can't optimise" without checking auxiliary space | 7 | 3 | 2026-09-29 |
| Omits sort terms from time complexity until challenged | 1 | 1 | 2026-10-01 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Long silence (7+ min) when stuck instead of thinking aloud | 33 | 10 | 2026-10-01 |
| Defends/asserts instead of tracing when asked to dry-run | 35 | 10 | 2026-10-01 |
| Asks for a hint instead of attempting the question posed | 7 | 3 | 2026-10-01 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 33 | 10 | 2026-10-01 |
| No code written within the coding-phase budget | 32 | 10 | 2026-10-01 |

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
