# DSA Weaknesses
Last updated: 2026-08-19

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Asks for constraints but can't translate them to a budget | 20 | 10 | 2026-08-19 |
| Doesn't proactively ask about input semantics (empty, duplicates) | 24 | 9 | 2026-08-19 |
| Misses free structural facts stated in the problem | 11 | 3 | 2026-08-18 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 32 | 9 | 2026-08-19 |
| Adopts an optimality principle without proving it | 11 | 6 | 2026-08-19 |
| Doesn't pin an endpoint to reduce a 2D count to 1D | 1 | 1 | 2026-08-19 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 87 | 10 | 2026-08-19 |
| Tests only the given examples, never a self-made input | 8 | 8 | 2026-08-19 |
| Off-by-one / ordering bugs (update-before-shrink, mark-on-pop) | 21 | 4 | 2026-08-18 |
| Binary search: lo=mid paired with a down-biased mid | 1 | 4 | 2026-08-17 |
| Integer overflow despite value constraints in hand | 1 | 1 | 2026-08-19 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't check own complexity against the constraint budget | 11 | 10 | 2026-08-19 |
| Doesn't state complexity unless explicitly asked | 13 | 5 | 2026-08-17 |
| States a complexity contradicted by his own code | 2 | 2 | 2026-08-19 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Long silence (7+ min) when stuck instead of thinking aloud | 21 | 10 | 2026-08-19 |
| Defends/asserts instead of tracing when asked to dry-run | 28 | 10 | 2026-08-19 |
| Fixes code silently without stating what changed | 1 | 1 | 2026-08-19 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 21 | 10 | 2026-08-19 |
| No code written within the coding-phase budget | 20 | 10 | 2026-08-19 |
| Correction lands after the round would have ended | 1 | 1 | 2026-08-19 |

## Derivation Questions
<!-- Updated by /derive-optimal-algorithm. Ran = times he invoked the question unprompted
     when it was the one that mattered. Missed = times it was the unlocking question and he never reached it. -->
| # | Question | Ran | Missed | Last Missed |
|---|---|---|---|---|
| Q1 | Write the brute force as a function signature | 3 | 3 | 2026-08-04 |
| Q2 | Name the repeated work | 8 | 1 | 2026-08-14 |
| Q3 | Fix the most constrained variable | 1 | 1 | 2026-08-19 |
| Q4 | Is the predicate monotone? | 2 | 4 | 2026-08-19 |
| Q5 | Which scan direction/order makes it known? | 2 | 1 | 2026-07-28 |
| Q6 | Name the operation, match the structure | 3 | 4 | 2026-08-17 |
| Q7 | Candidate set too small, or move set too small? | 1 | 4 | 2026-08-06 |
| Q8 | What is the minimal state? | 1 | 3 | 2026-08-17 |
| Q9 | Which constraint have I not spent? | 0 | 9 | 2026-08-19 |
