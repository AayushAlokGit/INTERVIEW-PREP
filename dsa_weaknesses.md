# DSA Weaknesses
Last updated: 2026-07-14 (Cherry Pickup II)

## Problem Understanding & Clarification
| Weakness | Sessions | Last Seen |
|---|---|---|
| Skips clarifying questions on value ranges / scale | 23 | 2026-07-08 |
| Asks for constraints but can't translate them to a budget | 3 | 2026-07-14 |
| Doesn't proactively ask about input semantics (negatives, sorted) | 7 | 2026-07-14 |
| Retreats into definitional questions when stuck | 1 | 2026-07-13 |
| Doesn't clarify strict vs non-strict / inclusive vs exclusive | 4 | 2026-06-30 |

## Approach & Thought Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 16 | 2026-07-13 |
| Applies generic template, misses problem-specific constraints | 9 | 2026-06-22 |
| Requests hint before attempting the sub-problem | 2 | 2026-07-13 |
| Never shrinks to smallest case (n=3) to find the insight | 1 | 2026-07-13 |
| Wrong binary-search anchor → non-monotonic predicate | 4 | 2026-05-26 |

## Code Quality & Correctness
| Weakness | Sessions | Last Seen |
|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 63 | 2026-07-14 |
| Off-by-one / boundary bugs (loop/empty/single-node/null) | 12 | 2026-07-14 |
| Writes reflexive code he can't justify (dead backtracking) | 1 | 2026-07-14 |
| Abandons problem before writing any code | 1 | 2026-07-13 |
| Tests masking input, not the boundary that breaks code | 9 | 2026-06-22 |

## Complexity Analysis
| Weakness | Sessions | Last Seen |
|---|---|---|
| Stated complexity doesn't match container used (map/string keys) | 7 | 2026-07-14 |
| Asserts best-case bound, omits true factor (descent/search) | 10 | 2026-07-06 |
| Wrong recursion/stack depth bound (ignores skewed worst case) | 6 | 2026-07-02 |
| Miscounts space: wrong param range or ignores aux buffers | 3 | 2026-07-13 |
| Checks overflow on one variable, ignores the accumulator | 1 | 2026-07-14 |

## Communication
| Weakness | Sessions | Last Seen |
|---|---|---|
| Drops one of multiple sub-questions asked | 8 | 2026-07-14 |
| Shallow edge case coverage — only one case volunteered | 23 | 2026-07-06 |
| Stops at "not sure" / abandons correct idea on obstacle | 19 | 2026-07-13 |
| Defends/asserts instead of tracing when asked to dry-run | 14 | 2026-07-14 |
| Traces intent not actual code (ignores own loop bounds) | 10 | 2026-07-14 |

## Time Management
| Weakness | Sessions | Last Seen |
|---|---|---|
