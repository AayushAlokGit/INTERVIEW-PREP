# DSA Weaknesses
Last updated: 2026-07-27 (Stone Game III)

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
| Defaults to generic pattern over structure-exploiting one | 18 | 2026-07-27 |
| Never audits recursion state; keeps params that change nothing | 2 | 2026-07-27 |
| Proposes greedy without justifying it against an adversary | 1 | 2026-07-27 |
| Guesses/revises recurrence instead of deriving from consumption | 1 | 2026-07-27 |
| Stops at correct-but-intractable brute force; asks for the fix | 1 | 2026-07-27 |

## Code Quality & Correctness
| Weakness | Sessions | Last Seen |
|---|---|---|
| Doesn't self-verify/dry-run before declaring done | 66 | 2026-07-27 |
| Off-by-one / boundary bugs (loop/empty/single-node/null) | 14 | 2026-07-23 |
| Abandons problem before writing any code | 3 | 2026-07-27 |
| States inequality/index direction backwards in prose vs code | 2 | 2026-07-27 |
| Mixes length-based vs position-based indexing conventions | 1 | 2026-07-27 |

## Complexity Analysis
| Weakness | Sessions | Last Seen |
|---|---|---|
| Stated complexity doesn't match container used (map/string keys) | 7 | 2026-07-14 |
| Asserts best-case bound, omits true factor (descent/search) | 10 | 2026-07-06 |
| Wrong recursion/stack depth bound (ignores skewed worst case) | 6 | 2026-07-02 |
| Proves invariant only within a loop, not across iterations | 1 | 2026-07-27 |
| Doesn't size the state space before committing to a recursion | 1 | 2026-07-27 |

## Communication
| Weakness | Sessions | Last Seen |
|---|---|---|
| Ignores a specific input handed to him; won't run the trace | 2 | 2026-07-27 |
| Long silence (10+ min) when stuck instead of thinking aloud | 4 | 2026-07-27 |
| Answers in fragments when asked for several specifics | 2 | 2026-07-27 |
| Stops at "not sure" / abandons correct idea on obstacle | 19 | 2026-07-13 |
| Defends/asserts instead of tracing when asked to dry-run | 17 | 2026-07-27 |

## Time Management
| Weakness | Sessions | Last Seen |
|---|---|---|
| Never reaches approach independently within budget | 4 | 2026-07-27 |
| No code written within the coding-phase budget | 3 | 2026-07-27 |
| Long silent coding stretch on an already-understood algorithm | 1 | 2026-07-27 |
