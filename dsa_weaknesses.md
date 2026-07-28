# DSA Weaknesses
Last updated: 2026-07-28 (Longest Substring With At Least K Repeating Characters)

## Problem Understanding & Clarification
| Weakness | Sessions | Last Seen |
|---|---|---|
| Skips clarifying questions on value ranges / scale | 25 | 2026-07-28 |
| Reads past a small bounded constant in constraints | 1 | 2026-07-28 |
| Doesn't proactively ask about input semantics (negatives, sorted) | 7 | 2026-07-14 |
| Asks for constraints but can't translate them to a budget | 3 | 2026-07-14 |
| Doesn't clarify strict vs non-strict / inclusive vs exclusive | 4 | 2026-06-30 |

## Approach & Thought Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 19 | 2026-07-28 |
| Proposes sliding window without checking predicate monotonicity | 1 | 2026-07-28 |
| Diagnoses a dead end but doesn't turn it into the next question | 1 | 2026-07-28 |
| Never audits recursion state; keeps params that change nothing | 2 | 2026-07-27 |
| Guesses/revises recurrence instead of deriving from consumption | 1 | 2026-07-27 |

## Code Quality & Correctness
| Weakness | Sessions | Last Seen |
|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 67 | 2026-07-28 |
| Off-by-one / boundary bugs (loop/empty/single-node/null) | 14 | 2026-07-23 |
| Uses standing condition (>=) where a crossing (==) is needed | 1 | 2026-07-28 |
| Abandons problem before writing any code | 3 | 2026-07-27 |
| States inequality/index direction backwards in prose vs code | 2 | 2026-07-27 |

## Complexity Analysis
| Weakness | Sessions | Last Seen |
|---|---|---|
| Doesn't state complexity unless explicitly asked | 1 | 2026-07-28 |
| Asserts best-case bound, omits true factor (descent/search) | 11 | 2026-07-28 |
| Stated complexity doesn't match container used (map/string keys) | 7 | 2026-07-14 |
| Wrong recursion/stack depth bound (ignores skewed worst case) | 6 | 2026-07-02 |
| Proves invariant only within a loop, not across iterations | 1 | 2026-07-27 |

## Communication
| Weakness | Sessions | Last Seen |
|---|---|---|
| Long silence (10+ min) when stuck instead of thinking aloud | 5 | 2026-07-28 |
| Defends/asserts instead of tracing when asked to dry-run | 17 | 2026-07-27 |
| Stops at "not sure" / abandons correct idea on obstacle | 19 | 2026-07-13 |
| Ignores a specific input handed to him; won't run the trace | 2 | 2026-07-27 |
| Answers in fragments when asked for several specifics | 2 | 2026-07-27 |

## Time Management
| Weakness | Sessions | Last Seen |
|---|---|---|
| Never reaches approach independently within budget | 5 | 2026-07-28 |
| Long silent coding stretch on an already-understood algorithm | 2 | 2026-07-28 |
| No code written within the coding-phase budget | 4 | 2026-07-28 |
