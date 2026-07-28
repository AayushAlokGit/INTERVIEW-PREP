# DSA Weaknesses
Last updated: 2026-07-28 (Burst Balloons)

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Skips clarifying questions on value ranges / scale | 27 | 10 | 2026-07-28 |
| Doesn't proactively ask about input semantics (negatives, zeros) | 10 | 10 | 2026-07-28 |
| Restates the problem instead of asking or attacking | 1 | 1 | 2026-07-28 |
| Never asks whether the input may be mutated | 1 | 1 | 2026-07-28 |
| Asks for constraints but can't translate them to a budget | 4 | 2 | 2026-07-28 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 20 | 10 | 2026-07-28 |
| Requests hints before writing down any candidate state | 1 | 1 | 2026-07-28 |
| Fails to recognise a recurrence he has already solved | 1 | 1 | 2026-07-28 |
| Asserts impossibility instead of arguing it | 1 | 1 | 2026-07-28 |
| Proposes sliding window without checking predicate monotonicity | 1 | 1 | 2026-07-28 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 69 | 10 | 2026-07-28 |
| Off-by-one / boundary bugs (loop/empty/single-node/null) | 14 | 7 | 2026-07-23 |
| Writes recurrence missing the boundary/coupling term | 1 | 1 | 2026-07-28 |
| States inequality/index direction backwards in prose vs code | 3 | 1 | 2026-07-28 |
| Uses standing condition (>=) where a crossing (==) is needed | 1 | 1 | 2026-07-28 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Asserts best-case bound, omits true factor (descent/search) | 11 | 8 | 2026-07-28 |
| Proves invariant only within a loop, not across iterations | 2 | 2 | 2026-07-28 |
| Doesn't state complexity unless explicitly asked | 2 | 2 | 2026-07-28 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defends/asserts instead of tracing when asked to dry-run | 18 | 9 | 2026-07-28 |
| Long silence (7+ min) when stuck instead of thinking aloud | 6 | 5 | 2026-07-28 |
| Answers in fragments when asked for several specifics | 3 | 1 | 2026-07-28 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 6 | 5 | 2026-07-28 |
| No code written within the coding-phase budget | 5 | 4 | 2026-07-28 |
