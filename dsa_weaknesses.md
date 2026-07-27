# DSA Weaknesses
Last updated: 2026-07-27 (132 Pattern)

## Problem Understanding & Clarification
| Weakness | Sessions | Last Seen |
|---|---|---|
| Skips clarifying questions on value ranges / scale | 24 | 2026-07-22 |
| Asks for constraints but can't translate them to a budget | 3 | 2026-07-14 |
| Doesn't proactively ask about input semantics (negatives, sorted) | 7 | 2026-07-14 |
| Asks for more examples instead of attacking the problem | 1 | 2026-07-22 |
| Doesn't clarify strict vs non-strict / inclusive vs exclusive | 4 | 2026-06-30 |

## Approach & Thought Process
| Weakness | Sessions | Last Seen |
|---|---|---|
| Defaults to generic pattern over structure-exploiting one | 17 | 2026-07-22 |
| Fixes wrong element; doesn't count residual query dimensions | 2 | 2026-07-27 |
| Requests hint/examples before attempting the sub-problem | 4 | 2026-07-27 |
| Never shrinks to smallest case (n=3) to find the insight | 2 | 2026-07-22 |
| Wrong binary-search anchor / wrong scan direction | 6 | 2026-07-27 |

## Code Quality & Correctness
| Weakness | Sessions | Last Seen |
|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 65 | 2026-07-23 |
| Off-by-one / boundary bugs (loop/empty/single-node/null) | 14 | 2026-07-23 |
| Abandons problem before writing any code | 2 | 2026-07-22 |
| States inequality direction backwards in prose vs code | 1 | 2026-07-27 |
| Tests masking input, not the boundary that breaks code | 9 | 2026-06-22 |

## Complexity Analysis
| Weakness | Sessions | Last Seen |
|---|---|---|
| Stated complexity doesn't match container used (map/string keys) | 7 | 2026-07-14 |
| Asserts best-case bound, omits true factor (descent/search) | 10 | 2026-07-06 |
| Wrong recursion/stack depth bound (ignores skewed worst case) | 6 | 2026-07-02 |
| Proves invariant only within a loop, not across iterations | 1 | 2026-07-27 |
| Miscounts space: wrong param range or ignores aux buffers | 3 | 2026-07-13 |

## Communication
| Weakness | Sessions | Last Seen |
|---|---|---|
| Long silence (8+ min) when stuck instead of thinking aloud | 2 | 2026-07-27 |
| Answers in fragments when asked for several specifics | 1 | 2026-07-27 |
| Stops at "not sure" / abandons correct idea on obstacle | 19 | 2026-07-13 |
| Defends/asserts instead of tracing when asked to dry-run | 16 | 2026-07-27 |
| Traces intent not actual code (ignores own loop bounds) | 10 | 2026-07-14 |

## Time Management
| Weakness | Sessions | Last Seen |
|---|---|---|
| Never reaches approach independently within budget | 2 | 2026-07-27 |
| No code written within the coding-phase budget | 2 | 2026-07-27 |
| Long silent coding stretch on an already-understood algorithm | 1 | 2026-07-27 |
