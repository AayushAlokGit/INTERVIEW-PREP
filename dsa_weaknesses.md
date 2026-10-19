# DSA Weaknesses
Last updated: 2026-10-19

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Asks for constraints but can't translate them to a budget | 31 | 9 | 2026-10-19 |
| Doesn't proactively ask about input semantics (sorted, duplicates) | 34 | 7 | 2026-10-19 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Adopts an optimality principle without proving it | 24 | 10 | 2026-10-19 |
| Defaults to generic pattern over structure-exploiting one | 44 | 9 | 2026-10-19 |
| Patches base cases per size instead of fixing definition | 1 | 1 | 2026-10-19 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Tests only the given examples, never a self-made input | 22 | 10 | 2026-10-19 |
| Doesn't self-verify/dry-run before declaring done | 99 | 10 | 2026-10-19 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't check own complexity against the constraint budget | 24 | 10 | 2026-10-19 |
| Declares "can't optimise" without checking state or space | 10 | 5 | 2026-10-19 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defends/asserts instead of tracing when asked to dry-run | 38 | 9 | 2026-10-19 |
| Long silence (7+ min) when stuck instead of thinking aloud | 33 | 6 | 2026-10-01 |
| Asks for a hint instead of attempting the question posed | 8 | 1 | 2026-10-19 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 36 | 10 | 2026-10-19 |
| No code written within the coding-phase budget | 33 | 8 | 2026-10-19 |

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
