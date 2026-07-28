# DSA Weaknesses
Last updated: 2026-07-28 (Longest Substring With At Least K Repeating Characters)

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Problem Understanding & Clarification
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Skips clarifying questions on value ranges / scale | 25 | 10 | 2026-07-28 |
| Doesn't proactively ask about input semantics (negatives, sorted) | 8 | 8 | 2026-07-28 |
| Asks for constraints but can't translate them to a budget | 4 | 4 | 2026-07-28 |
| Doesn't clarify strict vs non-strict / inclusive vs exclusive | 4 | 4 | 2026-06-30 |
| Reads past a small bounded constant in constraints | 1 | 1 | 2026-07-28 |

## Approach & Thought Process
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 19 | 10 | 2026-07-28 |
| Never audits recursion state; keeps params that change nothing | 2 | 2 | 2026-07-27 |
| Proposes sliding window without checking predicate monotonicity | 1 | 1 | 2026-07-28 |
| Diagnoses a dead end but doesn't turn it into the next question | 1 | 1 | 2026-07-28 |
| Guesses/revises recurrence instead of deriving from consumption | 1 | 1 | 2026-07-27 |

## Code Quality & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 67 | 10 | 2026-07-28 |
| Off-by-one / boundary bugs (loop/empty/single-node/null) | 14 | 9 | 2026-07-23 |
| Abandons problem before writing any code | 3 | 2 | 2026-07-27 |
| Uses standing condition (>=) where a crossing (==) is needed | 1 | 1 | 2026-07-28 |
| States inequality/index direction backwards in prose vs code | 2 | 1 | 2026-07-27 |

## Complexity Analysis
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Asserts best-case bound, omits true factor (descent/search) | 11 | 10 | 2026-07-28 |
| Stated complexity doesn't match container used (map/string keys) | 7 | 7 | 2026-07-14 |
| Wrong recursion/stack depth bound (ignores skewed worst case) | 6 | 6 | 2026-07-02 |
| Doesn't state complexity unless explicitly asked | 1 | 1 | 2026-07-28 |
| Proves invariant only within a loop, not across iterations | 1 | 1 | 2026-07-27 |

## Communication
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Defends/asserts instead of tracing when asked to dry-run | 17 | 9 | 2026-07-27 |
| Stops at "not sure" / abandons correct idea on obstacle | 19 | 9 | 2026-07-13 |
| Long silence (10+ min) when stuck instead of thinking aloud | 5 | 5 | 2026-07-28 |
| Ignores a specific input handed to him; won't run the trace | 2 | 1 | 2026-07-27 |
| Answers in fragments when asked for several specifics | 2 | 1 | 2026-07-27 |

## Time Management
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never reaches approach independently within budget | 5 | 5 | 2026-07-28 |
| No code written within the coding-phase budget | 4 | 4 | 2026-07-28 |
| Long silent coding stretch on an already-understood algorithm | 2 | 2 | 2026-07-28 |
