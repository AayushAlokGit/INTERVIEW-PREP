# DSA Weaknesses
Last updated: 2026-08-13

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't proactively ask about input semantics (empty, duplicates) | 19 | 10 | 2026-08-13 |
| Asks for constraints but can't translate them to a budget | 13 | 8 | 2026-08-13 |
| Misses free structural facts stated in the problem | 8 | 4 | 2026-08-13 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 27 | 10 | 2026-08-13 |
| Adopts an optimality principle without proving it | 7 | 5 | 2026-08-13 |
| Proposes sliding window without checking predicate monotonicity | 3 | 2 | 2026-08-05 |
| Patches the loop condition instead of shrinking the move set | 2 | 2 | 2026-08-13 |
| Picks nearest instead of earliest from own candidate set | 1 | 1 | 2026-08-13 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 80 | 10 | 2026-08-13 |
| Off-by-one / ordering bugs (update-before-shrink, mark-on-pop) | 18 | 4 | 2026-08-13 |
| Ignores integer overflow implied by the given constraints | 2 | 2 | 2026-08-06 |
| Misses degenerate/identity edge cases (source == target) | 2 | 1 | 2026-08-06 |
| Tests only the given examples, never a self-made input | 1 | 1 | 2026-08-13 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't state complexity unless explicitly asked | 11 | 8 | 2026-08-13 |
| Doesn't check own complexity against the constraint budget | 4 | 4 | 2026-08-13 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Long silence (7+ min) when stuck instead of thinking aloud | 16 | 10 | 2026-08-13 |
| Defends/asserts instead of tracing when asked to dry-run | 24 | 9 | 2026-08-13 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 16 | 10 | 2026-08-13 |
| No code written within the coding-phase budget | 15 | 10 | 2026-08-13 |

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
| Q7 | Candidate set too small, or move set too small? | 1 | 4 | 2026-08-06 |
| Q8 | What is the minimal state? | 1 | 2 | 2026-08-05 |
| Q9 | Which constraint have I not spent? | 0 | 5 | 2026-08-06 |
