# DSA Weaknesses
Last updated: 2026-08-14

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't proactively ask about input semantics (empty, duplicates) | 19 | 9 | 2026-08-13 |
| Asks for constraints but can't translate them to a budget | 14 | 9 | 2026-08-14 |
| Misses free structural facts stated in the problem | 9 | 5 | 2026-08-14 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 28 | 10 | 2026-08-14 |
| Adopts an optimality principle without proving it | 7 | 4 | 2026-08-13 |
| Proposes sliding window without checking predicate monotonicity | 3 | 2 | 2026-08-05 |
| Asks interviewer for the structural insight he half-saw | 1 | 1 | 2026-08-14 |
| Patches the loop condition instead of shrinking the move set | 2 | 1 | 2026-08-13 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 81 | 10 | 2026-08-14 |
| Off-by-one / ordering bugs (update-before-shrink, mark-on-pop) | 19 | 5 | 2026-08-14 |
| Tests only the given examples, never a self-made input | 2 | 2 | 2026-08-14 |
| Ignores integer overflow implied by the given constraints | 2 | 2 | 2026-08-06 |
| Writes a memo but never reads it | 1 | 1 | 2026-08-14 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't state complexity unless explicitly asked | 12 | 9 | 2026-08-14 |
| Doesn't check own complexity against the constraint budget | 5 | 5 | 2026-08-14 |
| States complexity of intended algorithm, not written code | 1 | 1 | 2026-08-14 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defends/asserts instead of tracing when asked to dry-run | 25 | 10 | 2026-08-14 |
| Long silence (7+ min) when stuck instead of thinking aloud | 17 | 10 | 2026-08-14 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 17 | 10 | 2026-08-14 |
| No code written within the coding-phase budget | 16 | 10 | 2026-08-14 |

## Derivation Questions
<!-- Updated by /derive-optimal-algorithm. Ran = times he invoked the question unprompted
     when it was the one that mattered. Missed = times it was the unlocking question and he never reached it. -->
| # | Question | Ran | Missed | Last Missed |
|---|---|---|---|---|
| Q1 | Write the brute force as a function signature | 3 | 3 | 2026-08-04 |
| Q2 | Name the repeated work | 6 | 1 | 2026-08-14 |
| Q3 | Fix the most constrained variable | 1 | 0 | — |
| Q4 | Is the predicate monotone? | 1 | 3 | 2026-08-05 |
| Q5 | Which scan direction/order makes it known? | 2 | 1 | 2026-07-28 |
| Q6 | Name the operation, match the structure | 1 | 3 | 2026-08-14 |
| Q7 | Candidate set too small, or move set too small? | 1 | 4 | 2026-08-06 |
| Q8 | What is the minimal state? | 1 | 2 | 2026-08-05 |
| Q9 | Which constraint have I not spent? | 0 | 5 | 2026-08-06 |
