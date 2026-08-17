# DSA Weaknesses
Last updated: 2026-08-17

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't proactively ask about input semantics (empty, duplicates) | 21 | 10 | 2026-08-17 |
| Asks for constraints but can't translate them to a budget | 16 | 10 | 2026-08-17 |
| Misses free structural facts stated in the problem | 10 | 5 | 2026-08-17 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 30 | 10 | 2026-08-17 |
| Adopts an optimality principle without proving it | 8 | 4 | 2026-08-17 |
| Commits to a DP state without testing it can distinguish cases | 1 | 1 | 2026-08-17 |
| Proves monotonicity, then picks the weaker tool it licenses | 1 | 1 | 2026-08-17 |
| Proposes sliding window without checking predicate monotonicity | 3 | 1 | 2026-08-05 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 83 | 10 | 2026-08-17 |
| Off-by-one / ordering bugs (update-before-shrink, mark-on-pop) | 20 | 6 | 2026-08-17 |
| Tests only the given examples, never a self-made input | 4 | 4 | 2026-08-17 |
| Binary search: lo=mid paired with a down-biased mid | 1 | 4 | 2026-08-17 |
| Ignores integer overflow implied by the given constraints | 3 | 3 | 2026-08-17 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't state complexity unless explicitly asked | 13 | 9 | 2026-08-17 |
| Doesn't check own complexity against the constraint budget | 7 | 7 | 2026-08-17 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defends/asserts instead of tracing when asked to dry-run | 26 | 10 | 2026-08-17 |
| Long silence (7+ min) when stuck instead of thinking aloud | 18 | 10 | 2026-08-17 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 18 | 10 | 2026-08-17 |
| No code written within the coding-phase budget | 17 | 10 | 2026-08-17 |

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
