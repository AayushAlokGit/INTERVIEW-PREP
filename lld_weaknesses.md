# LLD Weaknesses
Last updated: 2026-08-06

<!-- Sessions = lifetime count (never decreases). Active = current severity 0-10;
     -1 whenever a round gave the chance to exhibit it and he didn't. Row retires at Active 0. -->

## Requirements & Scoping
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Never produces an Out of Scope list | 1 | 1 | 2026-08-06 |
| Doesn't ask whether design is concurrent | 1 | 1 | 2026-08-06 |
| Doesn't ask the error/contract semantics | 1 | 1 | 2026-08-06 |
| Omits given rules from written requirements | 1 | 1 | 2026-08-06 |

## Entity Modelling
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| _(none observed)_ | | | |

## Class Design & Encapsulation
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Incomplete signatures on first submission | 1 | 1 | 2026-08-06 |
| Return type contradicts method body | 1 | 1 | 2026-08-06 |
| Two-call public API leaks check-then-act | 1 | 1 | 2026-08-06 |

## Responsibility Placement
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Entities enforce no invariants of their own | 1 | 1 | 2026-08-06 |
| Getters used so orchestrator decides (Ask) | 1 | 1 | 2026-08-06 |
| Policy rules land in the orchestrator | 1 | 1 | 2026-08-06 |

## Implementation & Correctness
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| No self-trace or edge cases before submitting | 1 | 1 | 2026-08-06 |
| Relies on background job for a live rule | 1 | 1 | 2026-08-06 |
| Trusts objects passed in by callers | 1 | 1 | 2026-08-06 |

## Simplicity & Patterns
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| _(none observed — restraint was a strength)_ | | | |

## Extensibility & Concurrency
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Won't interleave threads when asked | 1 | 1 | 2026-08-06 |
| Reaches for a lock without naming the category | 1 | 1 | 2026-08-06 |
| Duplicates data structures instead of generalising | 1 | 1 | 2026-08-06 |
| States no cost for a chosen primitive | 1 | 1 | 2026-08-06 |

## Communication & Pace
| Weakness | Sessions | Active | Last Seen |
|---|---|---|---|
| Asks requirements one per turn, not batched | 1 | 1 | 2026-08-06 |
| Overruns requirements phase (11 min vs 5) | 1 | 1 | 2026-08-06 |

## Senior Signals
| Signal | Status | Last Seen |
|---|---|---|
| Scopes before designing | Mixed | 2026-08-06 |
| State derived from requirements | Strong | 2026-08-06 |
| Rules live with their state (Tell, Don't Ask) | Weak | 2026-08-06 |
| Simplicity held under pressure | Strong | 2026-08-06 |
| Verifies own logic | Weak | 2026-08-06 |
| Extends without rewriting | Mixed | 2026-08-06 |
