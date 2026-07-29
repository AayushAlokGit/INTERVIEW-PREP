# DSA Weaknesses
Last updated: 2026-07-29 (derivation drill: Car Fleet, Making a Large Island, Max Sum Circular Subarray)

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't proactively ask about input semantics (empty, duplicates) | 12 | 10 | 2026-07-28 |
| Skips clarifying questions on value ranges / scale | 28 | 9 | 2026-07-23 |
| Asks for constraints but can't translate them to a budget | 6 | 4 | 2026-07-28 |
| Misses free structural facts stated in the problem | 2 | 2 | 2026-07-28 |
| Restates the problem instead of asking or attacking | 2 | 1 | 2026-07-28 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 22 | 10 | 2026-07-28 |
| Proposes sliding window without checking predicate monotonicity | 2 | 2 | 2026-07-28 |
| Fails to recognise a technique he has already used | 2 | 1 | 2026-07-28 |
| Tracks state that preprocessing could eliminate | 1 | 1 | 2026-07-28 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 71 | 10 | 2026-07-28 |
| Off-by-one / ordering bugs (update-before-shrink, boundaries) | 15 | 7 | 2026-07-28 |
| Increment and decrement guards not mirror conditions | 2 | 2 | 2026-07-28 |
| Writes recurrence missing the boundary/coupling term | 1 | 1 | 2026-07-28 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Asserts best-case bound, omits true factor (descent/search) | 11 | 6 | 2026-07-23 |
| Doesn't state complexity unless explicitly asked | 4 | 4 | 2026-07-28 |
| Proves invariant only within a loop, not across iterations | 3 | 3 | 2026-07-28 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defends/asserts instead of tracing when asked to dry-run | 19 | 9 | 2026-07-28 |
| Long silence (7+ min) when stuck instead of thinking aloud | 8 | 7 | 2026-07-28 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 8 | 7 | 2026-07-28 |
| No code written within the coding-phase budget | 7 | 6 | 2026-07-28 |

## Derivation Questions
<!-- Updated by /derive-optimal-algorithm. Ran = times he invoked the question unprompted
     when it was the one that mattered. Missed = times it was the unlocking question and he never reached it. -->
| # | Question | Ran | Missed | Last Missed |
|---|---|---|---|---|
| Q1 | Write the brute force as a function signature | 1 | 1 | 2026-07-28 |
| Q2 | Name the repeated work | 5 | 0 | — |
| Q3 | Fix the most constrained variable | 1 | 0 | — |
| Q4 | Is the predicate monotone? | 0 | 1 | 2026-07-28 |
| Q5 | Which scan direction/order makes it known? | 1 | 1 | 2026-07-28 |
| Q6 | Name the operation, match the structure | 0 | 1 | 2026-07-28 |
| Q7 | Candidate set too small, or move set too small? | 1 | 1 | 2026-07-28 |
| Q8 | What is the minimal state? | 0 | 0 | — |
| Q9 | Which constraint have I not spent? | 0 | 1 | 2026-07-29 |

## Adversarial Inputs
<!-- Updated by /derive-optimal-algorithm. One row per category, 3 attempts per session (one per problem).
     Hit = concrete input + correct predicted-vs-actual output. Weak = right danger, no concrete input
     or wrong prediction. Miss = nothing, or an input the algorithm handles fine. -->
| Category | Attempts | Hits | Weak | Miss | Last Miss |
|---|---|---|---|---|---|
| Degenerate (empty / single / all-identical) | 3 | 1 | 1 | 1 | 2026-07-29 |
| Assumption-breaker (negatives, ties, boundary) | 3 | 1 | 1 | 1 | 2026-07-29 |
| Counter desync (increment/decrement not inverses) | 3 | 0 | 0 | 3 | 2026-07-29 |
