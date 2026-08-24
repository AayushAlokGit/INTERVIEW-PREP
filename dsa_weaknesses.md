# DSA Weaknesses
Last updated: 2026-08-24

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't proactively ask about input semantics (sorted, duplicates) | 32 | 10 | 2026-08-24 |
| Asks for constraints but can't translate them to a budget | 27 | 9 | 2026-08-24 |
| Never probes the tie-break / output-ordering rule | 1 | 1 | 2026-08-22 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 40 | 9 | 2026-08-24 |
| Adopts an optimality principle without proving it | 17 | 9 | 2026-08-24 |
| Takes the problem's operation phrasing as the algorithm axis | 4 | 3 | 2026-08-24 |
| Can't reduce a brute force without being told what to fix | 5 | 3 | 2026-08-24 |
| Rejects a complexity without deriving what must replace it | 1 | 1 | 2026-08-24 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Tests only the given examples, never a self-made input | 14 | 10 | 2026-08-24 |
| Doesn't self-verify/dry-run before declaring done | 91 | 9 | 2026-08-24 |
| Binary search: lo=mid paired with a down-biased mid | 1 | 2 | 2026-08-17 |
| Guards a boundary on one line, forgets it on the next | 1 | 1 | 2026-08-24 |
| Declares a modulus but never reduces the accumulator/return | 1 | 1 | 2026-08-24 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't check own complexity against the constraint budget | 18 | 9 | 2026-08-24 |
| Declares "can't optimise" without checking auxiliary space | 6 | 5 | 2026-08-24 |
| Misstates own complexity (ignores sort / map log factors) | 2 | 1 | 2026-08-24 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Long silence (7+ min) when stuck instead of thinking aloud | 29 | 10 | 2026-08-24 |
| Defends/asserts instead of tracing when asked to dry-run | 31 | 7 | 2026-08-24 |
| Asks for a hint instead of attempting the question posed | 4 | 1 | 2026-08-24 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 29 | 10 | 2026-08-24 |
| No code written within the coding-phase budget | 28 | 10 | 2026-08-24 |

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
